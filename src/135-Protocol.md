# Protocol #135 (RSC)

This is a RuneScape Classic protocol.
You can find a partially refactored client for it [here](https://bitbucket.org/_mthd0/rsc/).

## Packet structure

```
if (len >= 160 {
    [byte] (160 + (len / 256))
    [byte] (len & 0xff)
} else {
    [byte] len
    if (len > 0) {
        len--; // skip it
        [byte] data[len] // last byte
    }
}
[byte] opcode
for (int i = 0; i < len; i++)
    [byte] data[i]
```

## Byte order

This protocol uses big-endian byte order exclusively.

## Data types

| Name | Description |
|---|---|
| int8 | An 8-bit integer (i.e. byte) |
| int16 | An 16-bit integer (i.e. short/Word) |
| int32 | An 32-bit integer (i.e. int/DWord) |
| int64 | An 64-bit integer (i.e. long/QWord) |

Note: If the name is postfixed with `u`, it means that the type is unsigned.

Note: If the name is postfixed with `a`, then when writing, increment the MSB
(most significant byte, or first byte) by 128.
When reading, if the first byte unsigned is lower than 128, that's your
value, otherwise decrement the first byte by 128 and read as normally.

## Bit access

```java
private static final int bitmasks[] = {
    0, 1, 3, 7, 15, 31, 63, 127, 255, 511, 1023, 2047, 4095, 8191, 16383, 32767, 65535,
    0x1ffff, 0x3ffff, 0x7ffff, 0xfffff, 0x1fffff, 0x3fffff, 0x7fffff, 0xffffff, 0x1ffffff,
    0x3ffffff, 0x7ffffff, 0xfffffff, 0x1fffffff, 0x3fffffff, 0x7fffffff, -1
};

public void start_bit_access() {
    bitpos = offset * 8;
}

public void write_bits(int num, int val) {
    int bytepos = bitpos >> 3;
    int bitoffset = 8 - (bitpos & 7);
    bitpos += num;
    for (; num > bitoffset; bitoffset = 8) {
        buffer[bytepos] &= ~bitmasks[bitoffset];
        buffer[bytepos++] |= (val >> (num - bitoffset)) & bitmasks[bitoffset];
        num -= bitoffset;
    }
    if (num == bitoffset) {
        buffer[bytepos] &= ~bitmasks[bitoffset];
        buffer[bytepos] |= val & bitmasks[bitoffset];
    } else {
        buffer[bytepos] &= ~(bitmasks[num] << (bitoffset - num));
        buffer[bytepos] |= (val & bitmasks[num]) << (bitoffset - num);
    }
}

public void end_bit_access() {
    offset = (bitpos + 7) / 8;
}
```

## Reference

Player usernames are encoded and decoded with the following methods:

```java
public static long encode_37(String s) {
    String s1 = "";
    for (int i = 0; i < s.length(); i++) {
        char c = s.charAt(i);
        if (c >= 'a' && c <= 'z')
            s1 = s1 + c;
        else if (c >= 'A' && c <= 'Z')
            s1 = s1 + (char) ((c + 97) - 65);
        else if (c >= '0' && c <= '9')
            s1 = s1 + c;
        else
            s1 = s1 + ' ';
    }
    s1 = s1.trim();
    if (s1.length() > 12)
        s1 = s1.substring(0, 12);
    long l = 0L;
    for (int j = 0; j < s1.length(); j++) {
        char c1 = s1.charAt(j);
        l *= 37L;
        if (c1 >= 'a' && c1 <= 'z')
            l += (1 + c1) - 97;
        else if (c1 >= '0' && c1 <= '9')
            l += (27 + c1) - 48;
    }
    return l;
}

public static String decode_37(long l) {
    String s = "";
    while (l != 0L) {
        int i = (int) (l % 37L);
        l /= 37L;
        if (i == 0) {
            s = " " + s;
        } else if (i < 27) {
            if (l % 37L == 0L)
                s = (char) ((i + 65) - 1) + s;
            else
                s = (char) ((i + 97) - 1) + s;
        } else {
            s = (char) ((i + 48) - 27) + s;
        }
    }
    return s;
}
```

Chat messages are encoded and decoded with the following methods:

```java
public static byte encodedmsg[] = new byte[100];
public static char decodedmsg[] = new char[100];
private static char charmap[] = {
    ' ', 'e', 't', 'a', 'o', 'i', 'h', 'n', 's', 'r', 'd', 'l', 'u', 'm', 'w', 'c', 'y', 'f',
    'g', 'p', 'b', 'v', 'k', 'x', 'j', 'q', 'z', '0', '1', '2', '3', '4', '5', '6', '7', '8',
    '9', ' ', '!', '?', '.', ',', ':', ';', '(', ')', '-', '&', '*', '\\', '\'', '@', '#', '+',
    '=', '\243', '$', '%', '"', '[', ']'
};

public static String decode_msg(byte buffer[], int off, int enclen) {
    try {
        int i = 0;
        int j = -1;
        for (int k = 0; k < enclen; k++) {
            int l = buffer[off++] & 0xff;
            int i1 = l >> 4 & 0xf;
            if (j == -1) {
                if (i1 < 13)
                    decodedmsg[i++] = charmap[i1];
                else
                    j = i1;
            } else {
                decodedmsg[i++] = charmap[((j << 4) + i1) - 195];
                j = -1;
            }
            i1 = l & 0xf;
            if (j == -1) {
                if (i1 < 13)
                    decodedmsg[i++] = charmap[i1];
                else
                    j = i1;
            } else {
                decodedmsg[i++] = charmap[((j << 4) + i1) - 195];
                j = -1;
            }
        }
        boolean flag = true;
        for (int j1 = 0; j1 < i; j1++) {
            char c = decodedmsg[j1];
            if (j1 > 4 && c == '@')
                decodedmsg[j1] = ' ';
            if (c == '%')
                decodedmsg[j1] = ' ';
            if (flag && c >= 'a' && c <= 'z') {
                decodedmsg[j1] += '\uFFE0';
                flag = false;
            }
            if (c == '.' || c == '!')
                flag = true;
        }
        return new String(decodedmsg, 0, i);
    } catch (Exception ex) {
        return "Cabbage";
    }
}

public static int encode_msg(String str) {
    if (str.length() > 80)
        str = str.substring(0, 80);
    str = str.toLowerCase();
    int enclen = 0;
    int i = -1;
    for (int j = 0; j < str.length(); j++) {
        char c = str.charAt(j);
        int k = 0;
        for (int l = 0; l < charmap.length; l++) {
            if (c != charmap[l])
                continue;
            k = l;
            break;
        }
        if (k > 12)
            k += 195;
        if (i == -1) {
            if (k < 13)
                i = k;
            else
                encodedmsg[enclen++] = (byte) k;
        } else if (k < 13) {
            encodedmsg[enclen++] = (byte) ((i << 4) + k);
            i = -1;
        } else {
            encodedmsg[enclen++] = (byte) ((i << 4) + (k >> 4));
            i = k & 0xf;
        }
    }
    if (i != -1)
        encodedmsg[enclen++] = (byte) (i << 4);
    return enclen;
}
```

## Login

* The username and password are prepared by replacing any spaces or
illegal characters with _ and appending spaces to the string until its
length is 20.
* The connection with the server is established.
* The client reads a raw long from the server, this is the "session id".
* The client creates a new frame, opcode 0 or 19 if the player is reconnecting.
* A short, the client's revision number (135) is placed in the buffer.
* A long, the player's username encoded with mod37 (see above) is placed in the
buffer.
* A string encoded with RSA and the player's session id (the password) is
placed in the buffer.
* An integer, the player's "ranseed" value is placed in the buffer.
* "ranseed" does not seed anything. RSC135 does not use ISAAC ciphering.
  It is an applet parameter or read from uid.dat. Presumably, it was
  used to identify players connecting from the same computer.
* The stream is then flushed.
* A byte is read from the stream and discarded.
* Another byte is read, this is the login response code from the server.

Login responses

| Response Code | Description
|---|---|
| 0 | Success. |
| 1 | ? |
| 3 | Invalid username or password. |
| 4 | Username already in use. |
| 5 | Client has been updated. |
| 6 | IP address already in use. |
| 7 | Login attempts exceeded. |
| 11 | Account temporarily disabled. |
| 12 | Account permanently disabled. |
| 15 | Server is currently full. |
| 16 | Members-only server. |
| 17 | Members only area? |

## Registration

* The username and password are prepared.
  This is done by replacing any spaces or illegal characters
  with \_ and appending spaces to the string until its length is 20.
* The connection with the server is established.
* The client reads a raw long from the server, this is the "session id".
* The client creates a new frame, opcode 2.
* A short, the client's revision number (135) is placed in the buffer.
* A long, the player's username encoded with mod37 (see above) is placed
  in the buffer.
* A short, the applet's "referid" parameter is placed in the buffer.
* A string encoded with RSA and the player's session id (the password)
is placed in the buffer.
* An integer, the player's "ranseed" value is placed in the buffer.
  "ranseed" does not seed anything. RSC135 does not use ISAAC ciphering.
  It is an applet parameter or read from uid.dat.
  Presumably, it was used to identify players connecting from the same computer.
* The stream is then flushed.
* A byte is read from the stream and discarded.
* Another byte is read, this is the newplayer response code from the server.

Registration responses

| Response Code | Description |
|---|---|
| 2 | Success. |
| 3 | Username already taken. |
| 5 | Client has been updated. |
| 6 | IP address already in use. |
| 7 | Registration attempts exceeded? |

## Mob Appearance Update

Firstly, create a new packet with the opcode 250.
Write the total number of expected updates (uint16).

Secondly, for each entity to update, write the (server) index of the
mob the update applies to (uint16), and the 'update type' (int8)
followed by whatever data that type expects.

### Update type 1: Public chat messages

* uint8 - The length of the chat message.
* string (raw) - The encoded chat message.

### Update type 2: Combat damage

* uint8 - The damage recieved.
* uint8 - That mob's 'current' hitpoints level.
* uint8 - That mob's 'base' hitpoints.

### Update type 3/4: Projectiles

The update type is 3 when the target is a NPC, or 4 is the target is a
player.

The standard magic projectile id is 1, and the standard ranged
projectile id is 2.

* uint16 - The projectile's id.
* uint16 - The (server) index of the projectile's target entity.

### Update type 5: Appearance

Hair style, body type, leg type, and colours are as they are sent by the
client's character design packet.

* uint16 - The player's status? Doesn't appear to do anything.
* int64 - The player's username encoded with mod37.
* uint8 - The size of the sub-update.
* uint8 - The player's hair style + 1, or 0 if they are wearing a helmet.
* uint8 - The player's body type + 1, or 0 if they are wearing a platebody.
* uint8 - The player's leg type + 1, or 0 if they are wearing legs.
* uint8 - The animation id + 1 (look in the client) of the player's offhand item or 0.
* uint8 - The animation id + 1 of the player's hand item or 0.
* uint8 - The animation id + 1 of the player's head item or 0.
* uint8 - The animation id + 1 of the player's body item or 0.
* uint8 - The animation id + 1 of the player's leg item or 0.
* uint8 - The animation id + 1 of the player's neck item or 0.
* uint8 - The animation id + 1 of the player's shoes or 0.
* uint8 - The animation id + 1 of the player's gloves or 0.
* uint8 - The animation id + 1 of the player's cape or 0.
* uint8 - The player's hair colour.
* uint8 - The player's top colour.
* uint8 - The player's leg colour.
* uint8 - The player's skin colour.
* uint8 - The player's combat level.
* uint8 - 1 if the player is skulled.

## Packets
The packet opcodes are unchanged from previous revisions, presumably
this was before the protocol was being regularly modified to deter the
developers of bots such as AutoRune.

The payload/structure is quite similar to most other RSC revisions.

### Incoming packets
'''TODO: Duelling stuff, 240''' {\|
class="wikitable" \|- ! scope="col" width="140px" \| Name ! scope="col"
width="50px" \| Opcode ! scope="col" width="350px" \| Payload !
scope="col" width="300px" \| Description \|- ! Display Message \| 8 \|\|
\* string - A raw string, the message. \| Informs the client of a line
to be printed in the in-game message box. Messages preceded by @que@ are
sent to the quest history box, messages preceded by @pri@ are sent to
the private chat history box. \|- ! Close Connection \| 9 \|\| \* None
\| Forces the client to log out. \|- ! Logout Failed \| 10 \|\| \* None
\| You can't log out now! \|- ! Initialize Friends List \| 23 \|\| \*
uint8 - The total number of players in the list. \* int64... - The
friend's username, encoded with mod37. \* uint8... - The world the
friend is logged in to. 0 indicates the player is logged out. \|
Initializes the player's friends list. Variable length. \|- ! Update
Friends List \| 24 \|\| \* int64 - The friend's username, encoded with
mod37. \* uint8 - 0 if the friend is logged in, 1 if the friend is
logged out. \| Informs the client that a friend has logged in/out or
that a new friend has been added to the list. \|- ! Initialize Ignore
List \| 26 \|\| \* uint8 - The total number of players in the list. \*
int64... - The friend's username, encoded with mod37. \| Initializes the
player's ignore list. Variable length. \|- ! Initialize Privacy Settings
\| 27 \|\| \* int8 - 0/1. Block public chat messages. \* int8 - 0/1.
Block private chat messages. \* int8 - 0/1. Block trade requests. \*
int8 - 0/1. Block duel requests. \| Initializes the player's privacy
settings. \|- ! Private Message \| 28 \|\| \* int64 - The sender's
username, encoded with mod37. \* string - The encoded message. \| Sends
a private message to the client. \|- ! Player Movement \| 255 \|\| \*
bits\[10\] - This player's x position. \* bits\[12\] - This player's y
position. \* bits\[4\] - This player's direction. \* bits\[8\] - The
number of players the client already knows about to be sent (but it
reads them in order?) \*\* bits\[1\]... - If the player has not moved &
does not need to be removed, 0 & don't send the next 2 lots of bits,
otherwise 1. \*\* bits\[1\]... - 1 if the player is to be removed. \*\*
bits\[3/4\]... - The player's direction. 4 bits with a value of 12 if
the player is to be removed, otherwise 3 bits. \* bits\[11\]... - The
new player's (server) index. \* bits\[5\]... - The new player's offset x
position. \* bits\[5\]... - The new player's offset y position. \*
bits\[4\]... - The new player's direction. \* bits\[1\]... - something
to do with c\>s 254? 0 \| Updates the position/movement of the client's
player and nearby players. Usually sent every game engine tick (600ms)
rather than when needed as with other packets. Offset positions are
this\_x - player\_x and are incremented by 32 if less than zero.
Variable length. \|- ! Ground Item Positions \| 254 \|\| \* uint16 - The
id of the item to update \* int8 - The x position of the item relative
to the player (item\_x - player\_x) \* int8 - The y position of the item
relative to the player (item\_y - player\_y) \| Updates the positions of
nearby ground items. if ((id & 0x8000) == 0), remove the item.
Therefore, if the server increments the id by 0x8000, the item will be
removed by the client. Variable length. \|- ! Object Positions \| 253
\|\| \* uint16 - The id of the object to update \* int8 - The x position
of the object relative to the player (object\_x - player\_x) \* int8 -
The y position of the object relative to the player (object\_y -
player\_y) \| Updates the positions of nearby objects. Variable length.
If the id is real fuckin' big, remove it. \|- ! Whole Inventory \| 252
\|\| \* uint8 - The number of items in the player's inventory. \*
uint16... - The item's id. If equipped, increment by 0x8000. \*
int32a... - The item's stack size. Only sent when the item is stackable.
\| Sends over the player's whole inventory. Variable length. \|- !
Player Appearance \| 250 \|\| \* See \[\[135 Protocol\#Mob Appearance
Update\|Mob Appearance Update\]\] \| Updates things to do with nearby
players that aren't related to movement. \|- ! Boundary Positions \| 249
\|\| \* uint16 - The id of the bound to update \* int8 - The x position
of the bound relative to the player (object\_x - player\_x) \* int8 -
The y position of the bound relative to the player (object\_y -
player\_y) \* int8 - The bound's direction \| Updates the positions of
nearby bounds. Variable length. If the id is real fuckin' big, remove
it. \|- ! NPC Movement \| 248 \|\| \* bits\[8\] - The number of NPCs the
client already knows about to be sent (but it reads them in order?).
\*\* bits\[1\]... - 1 if the NPC has moved, otherwise 0 & don't send the
next 2 lots of bits. \*\* bits\[1\]... - 1 if the NPC is to be removed,
otherwise 0. \*\* bits\[3/4\]... - The NPC's direction, 4 bits with a
value of 12 if the NPC is to be removed, otherwise 3 bits. \*
bits\[11\]... - The new NPC's (server) index. \* bits\[5\]... - The new
NPC's offset x position. \* bits\[5\]... - The new NPC's offset y
position. \* bits\[4\]... - The new NPC's direction. \* bits\[9\]... -
The new NPC's id. \| Updates the positions/movement of nearby NPCs.
Usually sent every game engine tick (600ms) rather than when needed as
with other packets. Offset positions are player\_x - npc\_x and are
incremented by 32 if less than zero. Variable length. \|- ! NPC
Appearance \| 247 \|\| \* See \[\[135 Protocol\#Mob Appearance
Update\|Mob Appearance Update\]\]. \| Updates things to do with nearby
NPCs that aren't related to movement. Only the first two update types
apply (NPCs cannot send projectiles or have changed sprites). NPC server
index is used instead of player server index, obviously. \|- ! Display
Dialog \| 246 \|\| \* uint8 - The total number of options. \* uint8... -
The length of the option string. \* string... - The option string. \|
Displays a NPC chat dialog. Variable length. \|- ! Hide Dialog \| 245
\|\| \* None \| Hides the NPC chat dialog. \|- ! Initialize Plane \| 244
\|\| \* uint16 - The player's server index. \* uint16 - The width of the
plane. (2304) \* uint16 - The height of the plane. (1776) \* uint16 -
The index of the plane. (player\_y / multiplier) \* uint16 - The plane
multiplier. (944) \| Initializes the plane. Sent when the player first
logs in, and when the player is teleported or moves up/down a height.
\|- ! All Skills \| 243 \|\| \* for (int i = 0; i \< skill\_count; i++)
\* uint8... - The skill's current level. \* for (int i = 0; i \<
skill\_count; i++) \* uint8... - The skill's base level. \* for (int i =
0; i \< skill\_count; i++) \* int32... - The skill's xp points. \* uint8
- The player's quest points. \| Updates all of the player's skills and
quest points. The 135 client reads 18 skills: Attack, Defense, Strength,
Hits, Ranged, Prayer, Magic, Cooking, Woodcutting, Fletching, Fishing,
Firemaking, Crafting, Smithing, Mining, Herblaw, Carpentry, Thieving.
\|- ! Equipment Bonuses \| 242 \|\| \* uint8 - The armour's bonus. \*
uint8 - The weapon's accuracy bonus. \* uint8 - The weapon's strength
bonus. \* uint8 - The magic bonus. \* unit8 - The prayer bonus. \|
Updates the player's equipment bonuses. Variable length. \|- ! Player
Death \| 241 \|\| \* None \| Displays the "Oh dear! You are dead..."
screen. \|- ! Environment \| 240 \|\| \* ? \| Doesn't appear to be
important when you have everything else. It might be useful to have
though, so why don't you be kind and document it? :) \|- ! Display
Character Design \| 239 \|\| \* None \| Displays the character design
interface. \|- ! Display Trade Offer \| 238 \|\| \* uint16 - The server
index of the player we are trading with. \| Displays the trade offer
interface. \|- ! Hide Trade \| 237 \|\| \* None \| Hides the trade offer
and confirm interfaces. \|- ! Update Trade Offer \| 236 \|\| \* int8 -
The number of items the other player has traded. \* uint16... - The
item's id. \* int32a... - The item's stack size. \| Updates the other
player's trade offer. \|- ! Other's Trade Status \| 235 \|\| \* int8 - 1
= yes, anything else = no \| Has the other player accepted the trade
offer? \|- ! Display Shop \| 234 \|\| \* uint8 - The number of items in
the shop. \* int8 - 1 if the shop is a general store. \* uint8 - This
shop's selling price modifier. \* uint8 - This shop's buying price
modifier. \* uint16... - The item's id. \* uint16... - The item's stack
size. \* uint8... - The item's price. \| Displays the shop interface.
Variable length. \|- ! Hide Shop \| 233 \|\| \* None \| Hides the shop
interface. \|- ! Our Trade Status \| 229 \|\| \* int8 - 1 = yes,
anything else = no \| Have we accepted the trade offer? \|- ! Init Game
Settings \| 228 \|\| \* int8 - Automatic camera rotation. 1 = enabled,
anything else is disabled. \* int8 - Single mouse button. 1 = enabled,
anything else is disabled. \* int8 - Sound effects. 1 = disabled,
anything else is enabled. \| Sets the player's gameplay settings. \|- !
Set Prayers \| 227 \|\| \* int8... - The prayer's status. 1 = enabled,
anything else is disabled. \| Sets the status of every prayer. Variable
length. \|- ! Set Quests \| 226 \|\| \* int8... - The quest's completion
status. 1 = completed, anything else is incomplete. \| Sets the player's
quest completion status. Variable length. \|- ! Display Bank \| 222 \|\|
\* uint8 - The number of items in the player's bank. \* uint8 - The
maximum number of items the player is allowed to store. \* uint16... -
The item's id. \* int32a... - The item's stack size. \| Displays the
bank interface. Variable length. \|- ! Hide Bank \| 221 \|\| \* None \|
Hides the bank interface. \|- ! Bank Update \| 214 \|\| \* uint8 - The
item's slot. \* uint16 - The item's id. \* int32a - The item's stack
size. 0 to remove. \| Updates/adds/removes a single item in the bank
interface to save bytes. \|- ! Single XP Update \| 220 \|\| \* uint8 -
The skill's id. \* int32 - The skill's xp. \| Updates a single skill's
XP to save bytes. \|- ! Update InvItem \| 213 \|\| \* uint8 - The item's
slot. \* uint16 - The item's id. Increment by 0x7fff to change stack
size. \* int32a - The item's stack size. May not be read. \| Adds a
single item, or changes the ID, or changes the stack size to save bytes.
If id / 32768 == 1, the item is equipped. \|- ! Remove InvItem \| 212
\|\| \* uint8 - The item's slot. \| Removes a single item from the
player's inventory to save bytes. \|- ! Single Skill Update \| 211 \|\|
\* uint8 - The skill's id. \* uint8 - The skill's current level. \*
uint8 - The skill's base level. \* int32 - The skill's experience
points. \| Updates a single skill to save bytes. \|- ! Play Sound \| 207
\|\| \* byte\[\] - The name of the sound. \| Plays a sound/audio file in
the client. \|- ! Open Character Design Panel \| 203 \|\| \* None. \|
Open the character design panel. \|- \|} === '''Outgoing Data''' ===
'''TODO: recovery questions''' {\| class="wikitable" \|- ! scope="col"
width="140px" \| Name ! scope="col" width="50px" \| Opcode ! scope="col"
width="350px" \| Payload ! scope="col" width="300px" \| Description \|-
! Login \| 0 \|\| \* See \[\[135 Protocol\#Login\|Login\]\] \| Logs the
player in. \|- ! Reconnect \| 19 \|\| \* See \[\[135
Protocol\#Login\|Login\]\] \| Reconnects the player after they are
disconnected. \|- ! Disconnect \| 1 \|\| \* None \| Sent after the
server sends Close Connection (opcode 9), possibly to notify the server
that the player is to be removed. \|- ! Newplayer (Registration) \| 2
\|\| \* See \[\[135 Protocol\#Registration\|Registration\]\] \|
Registers a new user. \|- ! Public Chat \| 3 \|\| \* String - The
encoded message. \| Sends a message to public chat. \|- ! Ping \| 5 \|\|
\* None \| Ping \|- ! Attempt Logout \| 6 \|\| \* None \| Inform the
server that the client is attempting to log out \|- ! Admin Command \| 7
\|\| \* String - The command \| Sends a command to the server to be
executed \|- ! Report Abuse \| 10 \|\| \* int64 - The mod37 encoded
username to report \| Sends an abuse report to the server \|- ! Change
Password \| 25 \|\| \* RSAString - Your old password + new password
encoded with RSA \* Old password substring -
RSAString.substring(0,20).trim() \* New password substring -
RSAString.substring(20, RSAString.length()).trim(); \| Is used to change
your password ingame \|- ! Add Friend \| 26 \|\| \* int64 - mod37
encoded username \| Adds a user to your friends list \|- ! Remove Friend
\| 27 \|\| \* int64 - The mod37 encoded username to report \| Removes a
user from your friends list \|- ! Private Message \| 28 \|\| \* int64 -
The mod37 encoded username to send the message to \* String - The
message, scrambed \| Sends a message to the specified user \|- ! Ignore
User \| 29 \|\| \* int64 - The mod37 encoded username to ignore \| Adds
a user to your ignore list \|- ! Remove Ignore \| 30 \|\| \* int64 -
mod37 encoded username \| Removes a user from your ignore list \|- !
Update Privacy Settings \| 31 \|\| \* int8 - 1 to block public chat \*
int8 - 1 to block private chat \* int8 - 1 to block trade requests \*
int8 - 1 to block duel requests \| Updates the privacy settings \|- !
Walk to Tile \| 255 \|\| \* int16 - (start\_x + area\_x). The initial
position. \* int16 - (start\_y + area\_y) \* int8... - (route\_x\[i\] -
start\_x) \* int8... - (route\_y\[i\] - start\_y) \| Variable length.
Walks to a tile. The number of steps can be calculated by dividing the
available data by 2. \|- ! Walk to Entity \| 215 \|\| \* The same as
255. \| Variable length. Walks to an entity. The number of steps can be
calculated by dividing the available data by 2. \|- ! Player Response \|
254 \|\| \* int16 - The number of players sent \* int16... - The
player's server index \* int16... - The player's status, as sent with
the appearance update packet \| Variable length. Informs the server of
players the client knows about after a positions/movement update packet.
\|- ! Drop Item \| 251 \|\| \* int16 - The slot of the item to drop \|
Drops the specified item on the ground \|- ! Cast on Item \| 220 \|\| \*
int16 - The slot of the item to cast a spell on \* int16 - The id of the
spell to cast \| Casts a spell (such as High Alchemy) on the specified
item \|- ! Use with Item \| 240 \|\| \* int16 - The slot of the first
item to use \* int16 - The slot of the second item to use \| Uses an
item in the player's inventory with another item in the player's
inventory \|- ! Remove Item \| 248 \|\| \* int16 - The slot of the item
to unequip \| Unequips the specified inventory item \|- ! Wear Item \|
249 \|\| \* int16 - The slot of the item to equip \| Equips the
specified inventory item \|- ! Item Command \| 246 \|\| \* int16 - The
slot of the item to use \| Buries, eats, etc the specified inventory
item \|- ! Select Option \| 237 \|\| \* int8 - The position of the
option in the dialog\_options array \| Selects an option in a NPC dialog
\|- ! Combat Style \| 231 \|\| \* int8 - The position of the combat
style in the list \| Sets the player's combat style. \* 0 - Controlled
\* 1 - Aggressive \* 2 - Accurate \* 3 - Defensive \|- ! Close Bank \|
207 \|\| \* None \| Informs the server that the player has closed the
banking interface. \|- ! Withdraw Item \| 206 \|\| \* int16 - The ID of
the item to withdraw \* int16 - The amount of the specified item to
withdraw \| Withdraws a single type of item from the player's bank. \|-
! Deposit Item \| 205 \|\| \* int16 - The ID of the item to deposit \*
int16 - The amount of the specified item to deposit \| Deposits a single
type of item into the player's bank. \|- ! Disable Prayer \| 211 \|\| \*
int8 - The ID of the prayer to disable \| Disables a prayer. \|- !
Enable Prayer \| 212 \|\| \* int8 - The ID of the prayer to enable \|
Enables a prayer. \|- ! Update Game Setting \| 213 \|\| \* int8 - The
setting type \* int8 - The setting value (1 or 0) \| Setting types: \* 0
- Camera angle mode (auto/manual) \* 1 - Number of mouse buttons (1/2)
\* 2 - Sound effects (off/on) \|- ! Confirm Trade \| 202 \|\| \* None \|
Confirms the trade offer. \|- ! Accept Trade \| 232 \|\| \* None \|
Accepts the trade offer. \|- ! Decline Trade \| 233 \|\| \* None \|
Declines the trade offer. \|- ! Trade Update \| 234 \|\| \* int8 - The
amount of traded items to send to the server \* int16... - The id of the
item \* int32... - The amount/stack size of the item \| Variable length.
Updates the trade offer. \|- ! Cast on GItem \| 224\* \|\| \* int16 -
The item's X coordinate \* int16 - The item's Y coordinate \* int16 -
The item's ID \* int16 - The spell's ID \| Casts a spell on an item on
the ground. \|- ! Use with GItem \| 250\* \|\| \* int16 - The item's X
coordinate \* int16 - The item's Y coordinate \* int16 - The item's ID
\* int16 - The inventory slot \| Uses an item in the player's inventory
with an item on the ground. \|- ! Take GItem \| 252\* \|\| \* int16 -
The item's X coordinate \* int16 - The item's Y coordinate \* int16 -
The item's ID \| Picks up an item on the ground. \|- ! Cast on Boundary
\| 223\* \|\| \* int16 - The bound's X coordinate \* int16 - The bound's
Y coordinate \* int8 - The bound's direction \* int16 - The spell's ID
\| Casts a spell on a boundary (or 'wall object'). \|- ! Use with
Boundary \| 239\* \|\| \* int16 - The bound's X coordinate \* int16 -
The bound's Y coordinate \* int8 - The bound's direction \* int16 - The
inventory slot \| Uses an item in the player's inventory with a boundary
(or 'wall object'). \|- ! Boundary Cmd 1 \| 238\* \|\| \* int16 - The
bound's X coordinate \* int16 - The bound's Y coordinate \* int8 - The
bound's direction \| Performs the primary action (usually 'open') on a
boundary (or 'wall object'). \|- ! Boundary Cmd 2 \| 229\* \|\| \* int16
- The bound's X coordinate \* int16 - The bound's Y coordinate \* int8 -
The bound's direction \| Performs the secondary action (usually 'close'
or 'picklock') on a boundary (or 'wall object'). \|- ! Cast on Object \|
222\* \|\| \* int16 - The object's X coordinate \* int16 - The object's
Y coordinate \* int16 - The spell's ID \| Casts a spell on an object.
\|- ! Use with Object \| 241\* \|\| \* int16 - The object's X coordinate
\* int16 - The object's Y coordinate \* int16 - The inventory slot \|
Uses an item in the player's inventory with an object. \|- ! Object Cmd
1 \| 241\* \|\| \* int16 - The object's X coordinate \* int16 - The
object's Y coordinate \| Performs the primary action on an object (for
example, 'mine'). \|- ! Object Cmd 2 \| 230\* \|\| \* int16 - The
object's X coordinate \* int16 - The object's Y coordinate \| Performs
the secondary action on an object (for example, 'prospect'). \|- ! Cast
on NPC \| 225\* \|\| \* int16 - The NPC's server index \* int16 - The
spell's ID \| Casts a spell on a non-player character. \|- ! Use with
NPC \| 243\* \|\| \* int16 - The NPC's server index \* int16 - The
inventory slot \| Uses an item in the player's inventory with a
non-player character. \|- ! Talk to NPC \| 245\* \|\| \* int16 - The
NPC's server index \| Starts talking to a non-player character. \|- !
Attack NPC \| 244\* \|\| \* int16 - The NPC's server index \| Starts
attacking a non-player character. \|- ! NPC Cmd 2 \| 195\* \|\| \* int16
- The NPC's server index \| Performs the secondary action on a
non-player character, usually 'pickpocket'. \|- ! Cast on Self \| 227
\|\| \* int16 - The spell's ID \| Cast a teleport or charge spell on the
local player. \|- ! Cast on Player \| 226\* \|\| \* int16 - The player's
server index \* int16 - The spell's ID \| Casts a spell on another
player. \|- ! Use with Player \| 219\* \|\| \* int16 - The player's
server index \* int16 - The inventory slot \| Uses an item (for example,
a Gnomeball, or a Christmas cracker) on another player. \|- ! Attack
Player \| 228\* \|\| \* int16 - The player's server index \| Starts
attacking another player. \|- ! Trade Player \| 235 \|\| \* int16 - The
player's server index \| Sends a trade request to another player. \|- !
Follow Player \| 214 \|\| \* int16 - The player's server index \| Starts
following another player. \|- ! Duel Player \| 204 \|\| \* int16 - The
player's server index \| Sends a duel request to another player. \|- !
RuntimeException \| 17 \|\| \* String - The text of the error. \| Sent
when the client throws an exception while processing data sent by the
server. \|- ! Confirm Duel Offer \| 198 \|\| \* None \| Confirms the
duel offer. \|- ! Accept Duel Offer \| 199 \|\| \* None \| Accepts the
duel offer. \|- ! Duel Settings \| 200 \|\| \* int8 - No retreating, 1
or 0 \* int8 - No magic, 1 or 0 \* int8 - No prayers, 1 or 0 \* int8 -
No weapons, 1 or 0 \| Updates the duel settings. \|- ! Duel Items \| 201
\|\| \* int8 - The total number of offered items \* int16... - Offered
item ID \* int32... - Offered item stack size \| Variable length.
Updates the stake. \|- ! Decline Duel Offer \| 203 \|\| \* None \|
Declines the duel offer. \|- ! Character Design \| 236 \|\| \* int8 -
The player's gender - 2=Female, 1=Male \* int8 - The player's hair style
\* int8 - The player's 'body type' - 4=Female, 1=Male \* int8 - The
player's 'leg type' - always 2 \* int8 - The player's hair colour \*
int8 - The player's top colour \* int8 - The player's leg colour \* int8
- The player's skin colour \* int8 - The player's class \| Submits the
player's chosen design when they log in for the first time. \* 0 -
Adventurer class \* 1 - Warrior class \* 2 - Wizard class \* 3 - Ranger
class \* 4 - Miner class \|} Notes:

* Opcodes marked with \* are preceded by Walk to Entity.
* When closing the duel confirm screen, it may send the decline trade packet,
  for some reason.
