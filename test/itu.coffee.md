    mocha = require 'mocha'
    should = require 'should'

    {EventEmitter} = require 'events'

    describe 'The byte-buffer', ->
      b = require '../src/byte-buffer'
      describe 'should have an empty buffer', ->
        b.should.have.property 'empty_buffer'
        b.empty_buffer.should.have.length 0

      describe 'should have a one-byte-buffer operation', ->
        b.should.have.property 'one_byte_buffer'

      describe 'and it should build a one-byte-buffer', ->
        example = b.one_byte_buffer 0xe9
        example.should.have.length 1
        example.readUInt8(0).should.equal 0xe9

      describe 'should have a three-byte-buffer operation', ->
        b.should.have.property 'three_byte_buffer'

      describe 'and it should build a three-byte-buffer', ->
        example = b.three_byte_buffer 0x123456
        example.should.have.length 3
        example.readUInt8(0).should.equal 0x56
        example.readUInt8(1).should.equal 0x34
        example.readUInt8(2).should.equal 0x12


    describe 'The ITU formatter', ->
      itu = require '../src/itu-format'
      {one_byte_buffer} = require '../src/byte-buffer'

      message_type_code =
        code: null
        encode: (p) ->
          one_byte_buffer p.message_type_code
        decode: (p,buf) ->
          p.message_type_code = buf.readUInt8 0
          len = 1

      bar1 =
        code: 0x64
        encode: (p) ->
          if p.bar1?
            one_byte_buffer p.bar1
        decode: (p,buf) ->
          p.bar1 = buf.readUInt8 0
          len = 1

      bar2 =
        code: 0x78
        encode: (p) ->
          if p.bar2?
            buf = new Buffer 2
            buf.writeUInt16LE p.bar2, 0
            buf
        decode: (p,buf) ->
          p.bar2 = buf.readUInt16LE 0
          len = 2

      str =
        code: 0xde
        encode: (p) ->
          if p.str?
            new Buffer p.str
        decode: (p,buf) ->
          p.str = buf.toString()

      str2 =
        code: 0xdd
        encode: (p) ->
          if p.str2?
            new Buffer p.str2
          else
            null
        decode: (p,buf) ->
          p.str2 = buf.toString()

      describe 'for a simple format', ->
        message =
          FOO:
            message_type_code: 0x42
            mandatory_fixed: [
              message_type_code
            ]
        {build,parse} = itu message

        it 'should build properly', (done) ->
          buf = build
            message_type_code: message.FOO.message_type_code

          buf.should.have.length 1
          buf.readUInt8(0).should.equal 0x42
          done()

      describe 'for a more complex format', ->
        message =
          BAR:
            message_type_code: 0x78
            mandatory_fixed: [
              message_type_code
            ]
            mandatory_variable: [
              str
            ]
        {build,parse} = itu message
        it 'should build properly', (done) ->
          buf = build
            message_type_code: message.BAR.message_type_code
            str: 'Hello world!' # 12 characters

          buf.should.have.length 1 + 1 + 1 + 12
          buf.readUInt8(0).should.equal 0x78
          buf.readUInt8(1).should.equal 1
          buf.readUInt8(2).should.equal 12
          buf.slice(3).toString().should.equal 'Hello world!'
          done()

      describe 'and for another one with optional parameter but no values', ->
        message =
          BAR2:
            message_type_code: 42
            mandatory_fixed: [
              message_type_code
              bar1
            ]
            mandatory_variable: [
              str
              bar2
              bar1
            ]
            optional: [
              str2
            ]
        {build,parse} = itu message

        it 'should build properly', (done) ->
          buf = build
            message_type_code: message.BAR2.message_type_code
            str: 'Hello world!' # 12 characters
            bar1: 123
            bar2: 0xbeef

          buf.should.have.length 2 + 4 + 13 + 2 + 3
          # Mandatory fixed
          buf.readUInt8(0).should.equal 42
          buf.readUInt8(1).should.equal 123
          # Pointers
          buf.readUInt8(2).should.equal 4
          buf.readUInt8(3).should.equal 16
          buf.readUInt8(4).should.equal 18
          buf.readUInt8(5).should.equal 0 # no optional parameters
          # Mandatory variable
          buf.readUInt8(6).should.equal 12
          buf.slice(7,19).toString().should.equal 'Hello world!'
          buf.readUInt8(19).should.equal 2
          buf.readUInt8(20).should.equal 0xef
          buf.readUInt8(21).should.equal 0xbe
          buf.readUInt8(22).should.equal 1
          buf.readUInt8(23).should.equal 123
          done()

        it 'should decode back', (done) ->
          orig_data =
            message_type_code: message.BAR2.message_type_code
            str: 'Hello world!' # 12 characters
            bar1: 123
            bar2: 0xdead
          buf = build orig_data

          data = parse buf
          data.should.eql orig_data
          done()

      describe 'and for one with optional parameters', ->
        message =
          BAR3:
            message_type_code: 142
            mandatory_fixed: [
              message_type_code
              bar1
            ]
            mandatory_variable: [
              str
            ]
            optional: [
              str2
              bar2
              str
            ]
        {build,parse} = itu message

        it 'should build properly', (done) ->
          buf = build
            message_type_code: message.BAR3.message_type_code
            bar1: 123
            str: 'Hello world!' # 12 characters
            str2: 'Is this long enough?' # 20 characters
            bar2: 0xbeef

          buf.should.have.length 2 + 2 + 13 + 22 + 4 + 14
          # Mandatory fixed
          buf.readUInt8(0).should.equal 142
          buf.readUInt8(1).should.equal 123
          # Pointers
          buf.readUInt8(2).should.equal 2
          buf.readUInt8(3).should.equal 14
          # Mandatory variable
          buf.readUInt8(4).should.equal 12
          buf.slice(5,17).toString().should.equal 'Hello world!'
          # Optional
          buf.readUInt8(17).should.equal str2.code
          buf.readUInt8(18).should.equal 20
          buf.slice(19,39).toString().should.equal 'Is this long enough?'
          buf.readUInt8(39).should.equal bar2.code
          buf.readUInt8(40).should.equal 2
          buf.readUInt8(41).should.equal 0xef
          buf.readUInt8(42).should.equal 0xbe
          buf.readUInt8(43).should.equal str.code
          buf.readUInt8(44).should.equal 12
          buf.slice(45,57).toString().should.equal 'Hello world!'
          done()

        it 'should decode back', (done) ->
          orig_data =
            message_type_code: message.BAR3.message_type_code
            bar1: 123
            str: 'Hello world!' # 12 characters
            str2: 'Is this long enough?' # 20 characters
            bar2: 0xdead
          buf = build orig_data

          data = parse buf
          data.should.eql orig_data
          done()
