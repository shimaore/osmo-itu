uild and parse an ITU message from/to data structure
=====================================================

    {empty_buffer,one_byte_buffer} = require './byte-buffer'

    module.exports = (message_by_name) ->

ITU messages might contain (in that order) a Mandatory Fixed-length part, a Mandatory Variable-length part, and an Optional part.

      part_names = ['mandatory_fixed','mandatory_variable','optional']

      message_by_id = {}
      for name, v of message_by_name
        ref = {name}
        for part in part_names
          ref[part] = v[part] if v[part]?
        message_by_id[v.message_type_code] = ref

      tools =

Build
-----

        build: (data) ->

          message_type_code = data.message_type_code
          if typeof message_type_code is 'string'
            spec = message_by_name[message_type_code]
          else
            spec = message_by_id[message_type_code]

          unless spec?
            console.error "Unknown message type #{message_type_code}"
            return

Encode each individual parameter (in bulk).

          parts = {}
          for p in part_names
            parts[p] =
              if spec[p]? and spec[p].length > 0
                spec[p].map (parameter) ->
                  parameter.encode(data) ? null
              else
                null

Build Mandatory Fixed section

          fixed_part = Buffer.concat parts.mandatory_fixed

Build Mandatory Variable-length section

Since we do not deal with LUDT and LUDTS, all pointers are one octet in length; the total length of the pointer array is the number of mandatory variable parameters, plus one if an optional section is requested.

          number_of_mandatory_pointers = spec.mandatory_variable?.length ? 0
          number_of_optional_pointers = if spec.optional? then 1 else 0
          number_of_pointers = number_of_mandatory_pointers + number_of_optional_pointers

          pointers = new Buffer number_of_pointers

          if parts.mandatory_variable?
            variable_parameters = []
            pointer_index = 0
            start_of_parameter = number_of_pointers

            for parameter in parts.mandatory_variable

              # Only octets, since no LUDT or LUDTS
              # Q.713 @ 2.3, pointers are relative to the MSB
              pointers.writeUInt8 start_of_parameter-pointer_index, pointer_index
              pointer_index++

              param = Buffer.concat [
                one_byte_buffer parameter.length
                parameter
              ]

              variable_parameters.push param
              start_of_parameter += param.length

            variable_part = Buffer.concat variable_parameters

          else
            variable_part = empty_buffer

Build Optional section

          if parts.optional?
            optional_parameters = []
            for parameter, i in parts.optional
              if parameter?
                optional_parameters.push Buffer.concat [
                  # `parameter name`
                  one_byte_buffer spec.optional[i].code
                  # `length indicator of parameter`
                  one_byte_buffer parameter.length
                  # `parameter`
                  parameter
                ]
            optional_part = Buffer.concat optional_parameters

          else
            optional_part = empty_buffer

Pointer to start of optional part

          # Pointer to start of optional part
          if number_of_optional_pointers is 1
            if optional_part.length > 0
              pointers.writeUInt8 start_of_parameter-pointer_index, pointer_index
            else
              pointers.writeUInt8 0, pointer_index

Concatenate the three sections to obtain the full message.

          Buffer.concat [
            fixed_part
            pointers
            variable_part
            optional_part
          ]

Parse an ITU message into a data structure
------------------------------------------

        parse: (buf) ->
          type = buf.readUInt8 0
          spec = message_by_id[type]
          unless spec?
            console.error "Unknown ITU message type #{type}"
            return

          response = {}
          offset = 0

Parse the Mandatory Fixed section

          for part in spec.mandatory_fixed
            len = part.decode response, buf.slice offset
            offset += len

Parse the Mandatory Variable-length section

          for part in spec.mandatory_variable
            pointer = offset + buf.readUInt8 offset
            offset++
            # Assume only octet length, no LUDT/LUDTS
            len = buf.readUInt8 pointer
            pointer++
            part.decode response, buf.slice pointer, pointer+len

The last pointer of the Mandatory-Variable section points to the start of the optional section

          optional_offset = buf.readUInt8 offset

If it is zero then there are no optional options to process.

          if optional_offset is 0
            return response

Parse the Optional section

          offset += optional_offset

          while offset < buf.length

            code = buf.readUInt8 offset++
            # Assume only octet length, no LUDT/LUDTS
            len = buf.readUInt8 offset++

            unless offset+len <= buf.length
              console.error "Option goes beyond end of packet"
              # FIXME: throw? return response so far? ..

            segment = buf.slice offset, offset+len

Locate the option amongst the ones available for that packet type.

            for part in spec.optional
              if part.code is code
                part.decode response, segment
            # FIXME: error if could not find a proper part

            offset += len

Return the structure (we don't deal with events in this module)

          return response
