"""
Translates messages sent to & received from a Chromecast in the protocol-buffers format.

version 0.1

See https://developers.google.com/protocol-buffers/docs/encoding?hl=en

Thanks to TheCrazyT for this very helpful gist : https://gist.github.com/TheCrazyT/11263599

"""


# Copyright (C) 2014-2016 Pat Carter
#
# This file is part of Stream2chromecast.
#
# Stream2chromecast is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
#
# Stream2chromecast is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with Stream2chromecast.  If not, see <http://www.gnu.org/licenses/>.



from struct import pack, unpack


# Sent messages

def format_field_id(field_no, field_type):
    """ returns a field number & type for packing into the message """
    
    return (field_no << 3) | field_type
    


def format_varint_value(int_value):
    """ returns a varint type integer from a python integer """
    
    varint_result = ""        
    
    while(int_value > 127):
        varint_result += pack("B", int_value & 127 | 128) 
        int_value >>= 7

    varint_result += pack("B", int_value & 127)  #  & 127 unnecessary?
    
    return varint_result
          


def format_int_field(field_number, field_data):
    """ formats a protocol buffers Int field """
    
    field =  pack("B", format_field_id(field_number, 0))   #  0 = Int field type    
    field += pack("B", field_data)  
    
    return field 
    


def format_string_field(field_number, field_data):
    """ formats a protocol buffers length-delimited field """
    
    field_data_len = format_varint_value(len(field_data))
    
    field =  pack("B", format_field_id(field_number, 2))   #  2 = Length-delimited field type  
    field += pack("%ds" % len(field_data_len), field_data_len)
    field += pack("%ds" % len(field_data), field_data)   
    
    return field  
    
    
    
def prepend_length_header(msg):
    """ prepends the message with a length value """
    
    return pack(">I%ds" % len(msg), len(msg), msg)
    
    
    
def format_message(source_id, destination_id, namespace, data):    
    """ formats a message to be sent to the Chromecast """
    
    msg = ""
    msg += format_int_field(1, 0)   # Protocol Version  =  0
    msg += format_string_field(2, source_id)
    msg += format_string_field(3, destination_id)
    msg += format_string_field(4, namespace)
    msg += format_int_field(5, 0)   # payload type : string  =  0
    msg += format_string_field(6, data)
    
    msg = prepend_length_header(msg)        
    
    return msg
    
    
    

# Received messages
    
def extract_length_header(msg):
    """ extracts the length header from the first 4 bytes of a received message """
    
    if len(msg) < 4:
        print "!! Message too short"
        return None
        
    len_data = msg[:4]
    remainder = msg[4:]
    
    length = unpack(">I", len_data)[0]

    return length, remainder
    
    
    
def extract_varint(data):
    # extract varint
    # LSB first
    
    value = 0
    ptr = 0
    mul = 1

    while True:
        byte = unpack("B", data[ptr])[0]
        value += mul * (byte & 127)
        ptr += 1
        if not byte & 128:
            break

        mul *= 128

    remainder = data[ptr:]

    return value, remainder

def extract_field_id(data):
    """ extracts a field id from a received message """

    value, data = extract_varint(data)

    return (value >> 3, (value & 7)), data
    
    
    
def extract_int_field(data):
    """ extracts a protocol buffers Int field from a received message """
    
    field_id, data = extract_field_id(data)
    assert field_id[1] == 0 #1=64bit, 5=32bit, 0=varint

    #int_value = unpack("B", data[0])[0]
    #remainder = data[1:]
    int_value, remainder = extract_varint(data)
    
    return field_id, int_value, remainder
    
    
    
def extract_string_field(data):
    """ extracts a protocol buffers length-delimited field from a received message """
    
    field_id, data = extract_field_id(data)
    assert field_id[1] == 2
    
    length, data = extract_varint(data)
    string = data[:length]
    remainder = data[length:]

    #if length > len(string):
    #    print "Stream too short for string, got", len(string), "of", length, "bytes."

    return field_id, string, remainder
    
    
    
def extract_message(data):
    """ extracts the message data from a Chromecast response message """
    
    resp = {}
    
    field_id, resp['protocol'], data = extract_int_field(data)
    field_id, resp['source_id'], data = extract_string_field(data)
    field_id, resp['destination_id'], data = extract_string_field(data)
    field_id, resp['namespace'], data = extract_string_field(data)
    field_id, resp['payload_type'], data = extract_int_field(data)
    field_id, resp['data'], data = extract_string_field(data)

    #if data:
    #    print "Unexpected data at end of stream"
    
    return resp

    
