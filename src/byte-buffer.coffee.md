    module.exports =
      empty_buffer: new Buffer 0

      one_byte_buffer: (v) ->
        data = new Buffer 1
        data.writeUInt8 v, 0
        data

      three_byte_buffer: (v) ->
        # TODO: add parsing for 'xx:xx:xx' etc.
        data = new Buffer 4
        data.writeUInt32LE v, 0
        data.slice 0, 3
