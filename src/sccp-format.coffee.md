SCCP Format
===========

    {empty_buffer,one_byte_buffer,three_byte_buffer} = require './byte-buffer'

TS 48.006 @ 9.4.3

    discrimination:
      dtap: 1
      bssmap: 0

The user data field is found in the UDT message.

    parse_user_data_field: (buf) ->
      switch buf.readUInt8 0
        when 0: # bssmap
          len = buf.readUInt8 1
          msg = buf.splice 2, 2+len
          parse_bssmap msg

        when 1:
          dlci = buf.readUInt8 1
          len = buf.readUInt8 2
          msg = buf.splice 3, 3+len
          parse_dtap dlci, msg
          # Parsing of the DLCI, see 48.006 @ 9.3.2



Parameters
----------

We only include parameters relevant to TS 48.006.

    message_type_code =
      code: null # never optional nor variable length
      encode: (p) ->
        if typeof p.message_type_code is 'string'
          p.message_type_code = sccp_message_by_name[p.message_type_code].message_type_code
        one_byte_buffer p.message_type_code
      decode: (p,buf) ->
        p.message_type_code = buf.readUInt8 0
        len = 1

    end_of_optional_parameters = # Q.713 @ 3.1
      code: 0
      encode: (p) ->
        empty_buffer # null maybe? does Buffer.concat support it?

    subsystems = # Q.713 @ 3.4.2.2
      unknown: 0
      management: 1
      isup: 3
      omap: 4
      map: 5
      hlr: 6
      vlr: 7
      msc: 8
      eic: 9
      auc: 10
      iss: 11

    party_address = (v) -> # Q.713 @ 3.4
      # No GTT in BSC <-> MSC interface
      indicator = 0
      content = []

      if v.point_code # Q.713 @ 3.4.2.1
        point_code = new Buffer 2
        point_code.writeUInt16LE v.point_code
        content.push point_code
        indicator |= 1

      if v.subsystem # Q.713 @ 3.4.2.2
        if typeof v.subsystem is 'string'
          v.subsystem = subsystems[v.subsystem]
        subsystem = new Buffer [v.subsystem]
        content.push subsystem
        indicator |= 2

      # No GT support (skip Q.713 @ 3.4.23)

      # Routing indicator
      # Since we do not support GT, Route on SSN;
      # However only include this if a SSN is provided.
      if v.subsystem
        indicator |= 1 << 6 # Route on SSN

      content.unshift new Buffer [indicator]

      return Buffer.concat content

    destination_local_reference =
      code: 1
      encode = (p) ->
        three_byte_buffer p.destination_local_reference

    source_local_reference =
      code: 2
      encode: (p) ->
        three_byte_buffer p.source_local_reference

    called_party_address =
      code: 3
      encode: (p) ->
        party_address p.called_party_address

    calling_party_address =
      code: 4
      encode: (p) ->
        party_address p.called_party_address

    protocol_class =
      code: 5
      encode: (p) ->
        assert p.protocol_class is 0 or p.protocol_class is 2 or p.protocol_class is 128
        # 128 indicates "return message on error" for class 0
        new Buffer [p.protocol_class]

    segmenting_reassembling =
      code:
      encode: (p) ->
        new Buffer [ if p.segmenting_reassembling then 1 else 0 ]

    # Actually unused ( TS 48.006 @ 8.4.1 )
    credit =
      code:
      encode: (p) ->
        new Buffer [ p.credit ]

    from_table = (code,field,table) ->
      code:
      encode: (p) ->
        cause = p[field]
        if typeof cause is 'string'
          cause = table[cause]
        one_byte_buffer cause

    release_cause_code =
      end_user_originated: 0
      end_user_congestion: 1
      end_user_failure: 2
      sccp_user_originated: 3
      remote_procedure_error: 4
      inconsistent_connection_data: 5
      access_failure: 6
      access_congestion: 7
      subsystem_failure: 8
      subsystem_congestion: 9
      mtp_failure: 10
      network_congestion: 11
      expiration_of_reset_timer: 12
      expiration_of_receive_inactivity_timer: 13
      unqualified: 15
      sccp_failure: 16

    refusal_cause_code =
      end_user_originated: 0
      end_user_congestion: 1
      end_user_failure: 2
      sccp_user_originated: 3
      destination_address_unknown: 4
      destination_inaccessible: 5
      qos_not_available_non_transient: 6
      qos_not_available_transient: 7
      access_failure: 8
      access_congestion: 9
      subsystem_failure: 10
      subsystem_congestion: 11
      expiration_of_the_connection_establishment_timer: 12
      incompatible_user_data: 13
      unqualified: 15
      hop_counter_violation: 16
      sccp_failure: 17
      unequipped_user: 19

    release_cause = from_table --, 'release_cause', release_cause_code

    refusal_cause = from_table --, 'refusal_cause', refusal_cause_code

    data =
      code: 
      encode: (p) ->
        p.data # should already be a buffer

    # segmentation =  Unused?

    hop_counter = # Q.713 @ 3.18
      code:
      encode: (p) ->
        one_byte_buffer p.hop_counter

    importance = # Q.713 @ 3.19
      code:
      encode: (p) ->
        one_byte_buffer p.importance & 0x07

Message Type Codes
------------------

Q.713 table 1, minus messages not needed in 48.006

    sccp_message_by_name =
      CR: # Q.713 @ 4.2
        message_type_code: 1
        mandatory_fixed: [
          message_type_code
          source_local_reference
          protocol_class
        ]
        mandatory_variable: [
          called_party_address
        ]
        optional: [
          credit
          calling_party_address
          data
          hop_counter
          importance
          end_of_optional_parameters
        ]

      CC: # Q.713 @ 4.3
        message_type_code: 2
        mandatory_fixed: [
          message_type_code
          destination_local_reference
          source_local_reference
          protocol_class
        ]
        optional: [
          credit
          called_party_address
          data
          importance
          end_of_optional_parameters
        ]

      CREF: # Q.713 @ 4.4
        message_type_code: 3
        mandatory_fixed: [
          message_type_code
          destination_local_reference
          refusal_cause
        ]
        optional: [
          called_party_address
          data
          importance
          end_of_optional_parameters
        ]

      RLSD: # Q.713 @ 4.5
        message_type_code: 4
        mandatory_fixed: [
          message_type_code
          destination_local_reference
          source_local_reference
          release_cause
        ]
        optional: [
          data
          importance
          end_of_optional_parameters
        ]

      RLC: # Q.713 @ 4.6
        message_type_code: 5
        mandatory_fixed: [
          message_type_code
          destination_local_reference
          source_local_reference
        ]

      DT1: # Q.713 @ 4.7
        message_type_code: 
        mandatory_fixed: [
          message_type_code
          destination_local_reference
          segmenting_reassembling
        ]
        mandatory_variable: [
          data
        ]

      UDT: # Q.713 @ 4.10
        # where is return_cause, Q.713 @ 3.12 ?
        message_type_code: 9
        mandatory_fixed: [
          message_type_code
          protocol_class
        ]
        mandatory_variable: [
          called_party_address
          calling_party_address
          data
        ]

    module.exports = (require './itu-format') sccp_messages_by_name
