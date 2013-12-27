Parse SCCP
----------

    {build,parse} = request './sccp-format'

    module.exports = class sccpProtocol extends EventEmitter

      constructor: (@connection) ->
        @connection.on 'protocol sccp', (data) =>
          @parse data

      send: (buf,cb) ->
        @connection.send frame, cb

      parse: (data) ->

        if data.length < 1
          console.length "SCCP parser: data is too short"

        sccp = parse data
        if sccp?
          @connection.emit "sccp #{sccp.name}", sccp
        else
          @connection.emit 'error', 'Invalid SCCP frame'
