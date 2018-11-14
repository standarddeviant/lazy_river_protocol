# Lazy River Protocol v0.1

The Lazy River Protocol is aimed at standardizing streaming of multichannel sensor data using UDP and TCP. It is intended to simplify streaming of data between microcontrollers and other computers.

There are two main pieces of the Lazy River Protocol
    * `Advertisement` Data Format
    * `Payload Stream` Data Format

The `Advertisement` is a UDP packet that contains the relevant information to obtain the `Payload Stream` and potentially know the following metadata about the `Payload Stream`:
    * `fs` or sample rate
    * `num_chans` or number of channels
    * OPTIONALLY `chan_names` or names of the channels
    * OPTIONALLY `chan_units` or units of the channels

### Advertisement Packet Format
* [ 8 Byte Lazy River Magic Number ]
    * bytes 1:4 are ASCII chars of `UInt8['L', 'Z', 'B', 'C']`
    * bytes 5:8 are a `float32_t` where value = `42.0f`
        * This float informs the receiving side if endian-ness needs to be switched
        * The sender should package floats, including this one, in a consistent way

* [ N*4 Byte ID Name or Number ]
    * If first byte is 0x00, then ID is 24 bit, `uint24_t` of next 3 bytes
    * If first byte is not 0x00, then the ID is a null terminated ASCII string
        String is padded to a 4 byte boundary

* [ 4 Byte Generic Config ]
    * each bit is marked as
        * "don't care", expressed as `.`
        * default=0, expressed as `0`
        * default=1, expressed as `1`
    * byte 1 of 4: `0b000.....`
        * bit 1 of 8, "IPv4/IPv6 bit"
            * `0` means that ip is `IPv4` (default)
            * `1` means that ip is `IPv6`
        * bit 2 of 8, "TCP/UDP bit"
            * `0` means that stream is `TCP` (default)
            * `1` means that stream is `UDP`
        * bit 3 of 8, "ASCII/UTF8 bit"
            * `0` means that strings are `ASCII` (default)
            * `1` means that strings are `UTF-8`
    * byte 2 of 4: `0b........`
    * byte 3 of 4: `0b........`
    * byte 4 of 4: `0b........`

* [ 4 or 6 Byte TCP Serving `IP Address` ]
    * `IPv4` vs. `IPv6` is determined by Generic Config

* [ 2 Byte TCP `Serving Port` ]
    * `uint16_t`

* [ 4 Byte `Sample Rate` ]
    * `float32_t`

* [ 2 Byte Channel Count ]
    * `uint16_t`

* OPTIONAL: [ N*4 Byte `Channel Name`s ]
    * bytes 1:4 are ASCII chars of `UInt8['N', 'A', 'M', 'E']`
    * There will be concatenated, null-terminated strings, padded to a 4 byte boundary
    * This section is over when 2 sequential 0 bytes are found

* OPTIONAL: [ N*4 Byte `Unit String`s ]
    * bytes 1:4 are ASCII chars of `UInt8['U', 'N', 'I', 'T']`
    * All channel names are null terminated and then concatened
    * This concatenated string is padded to a 4 byte boundary with zeros
    * i.e. there are 2 channels, of `Volts` and `Ohms`, the following options are reasonable
        * `V\0Ohms\0\0` ( 8 byte string )
        * `Volts\0Ohms\0\0` ( 12 byte string)

* OPTIONAL [ N*4 Byte, `Unit Scalar Logarithm` ]
    * bytes 1:4 are ASCII chars of `UInt8['S', 'C', 'A', 'L']`
    * [ 4 Byte logarithm of base unit scalar ]
        * `float32_t`
        * i.e. if the actual units are millivolts or 10^(-3) volts, then the log10 scalar would be `-3.0f`

* OPTIONAL [ N*2 Byte, `Data Type` ]
    * bytes 1:4 are ASCII chars of `UInt8['D', 'T', 'Y', 'P']`
    * the data-type or `dtype` of each channel is described by a single byte
        * bit 1 is the 'complex bit' and signifies that complex data is being transported
            * This is important because it effectively doubles memory usage by a channel
        * bit 2 is a don't care        
        * bits 3:4 are a 2-bit lookup to describe the number type:
            * `0` means `unsigned int`
            * `1` means `signed int`
            * `2` means `floating point`
            * `3` is invalid
        * bits 5:8 are a `uint4` equal to log2(b) where b is the number of bits for each data element
    * Some 2-Byte examples of dtypes are
        * `float32_t` : `0b00100010 = 0x22`
        * `int16_t` : `0b00010001 = 0x11`
        * complex `float128_t` : `0b10100100 = 0xA4`
        * complex `int8_t` : `b10010000 = 0x90`
        * `uint8_t` : `b00000000 = 0x00`

### Payload Packet Format
* [ 8 Byte Lazy River Magic Number ]
    * bytes 1:4 are ASCII chars of `UInt8['L', 'Z', 'P', 'L']`
    * bytes 5:8 are a `float32_t` where value = `42.0f`
        * This float informs the receiving side if endian-ness needs to be switched
        * The sender should package floats, including this one, in a consistent way

* [ 2 Byte `uint16_t` of Channels ]
    * This is redundant from the advertisement, but here for simplicity

* [ 2 Byte `uint16_t` of Frames ]
    * A single Frame consists of 1 Sample from each Channel
        * This terminology is borrowed from .wav file specifications

* [ FIXME add a time-stamp format, NTP ??? ]

* [ N Byte Payload ]
    * This payload is heavily described by the Advertisement
    * Bytes making up the correct Data Types are placed here sequentially
