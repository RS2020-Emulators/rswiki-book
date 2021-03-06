\[\[Category Packet\]\] \[\[Category Packet 377\]\]
{{packet\|name=Construct Map Region\|description=Creates a map
region.\|opcode=53\|type=Variable Short\|length=N/A\|revision=377}} ==
Interface Animation ==

=== Description === Sets the animation for a model on an interface.

=== Packet Structure ===

{\| border=2 ! Data type ! Description \|- \| \[\[Data Types\#Non
Standard Data Types\|Big Endian\]\] \[\[Data Types\#Standard data
types\|Short\]\] \[\[Data Types\#Non Standard Data Types\|Special A\]\]
\| Map Region Y (absolute Y coordinate / 8 + 6) \|- \| \[\[Data
Types\#Standard data types\|Bit Block\]\] \| 1 bit (0 or 1) to decide if
a tile exists. 26 bits for data about the tile (only if it exists) \|-
\| \[\[Data Types\#Non Standard Data Types\|Big Endian\]\] \[\[Data
Types\#Standard data types\|Short\]\] \[\[Data Types\#Non Standard Data
Types\|Special A\]\] \| Map Region X (absolute X coordinate / 8 + 6) \|-
\|}

=== Information ===

If the tile exists then a 1 gets written to the output stream as a 1
otherwise a zero gets written to the stream.

If the tile does exist then you would follow that 1 bit with 26 bits of
information about the tile.

int info = inStream.readBits(26); int heightLevel = info \>\> 24 & 3;
int rotation = info \>\> 1 & 3; int regionX = info \>\> 14 & 0x3ff; int
regionY = info \>\> 3 & 0x7ff;

=== Implementation ===

for(int z = 0; z \< 4; z++) { for(int x = 0; x \< 13; x++) { for(int y =
0; y \< 13; y++) { outStream.writeBit(1, tileExists ? 1 : 0);
if(tileExists) outStream.writeBits(26, TILE\_INFORMATION); } } }
