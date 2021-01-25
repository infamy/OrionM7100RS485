# OrionM7100RS485
Effort to decode the M7100 RS485 head control protocol

# PHY Layer
PHY layer is pretty straight forward its rs485, 38400baud, with what seems to be a seperate ctrl/priority line used to indicate if the bus is free.

#Data Layer

Some simple data captures shows a pretty easy format. 

Src 1001 Dst counter msg 1017 checksum 1016 dst 
96 1001 E1 FC 0000 1B 5B48564F4C203D203238 1017 860F 1006 E1 96
This will display "VOL = 28" on the second line of the LCD.
Decoding the msg portion is very straight forward. The checksum is a different story.

So far i have had very little luck trying standard CRC used by protocols like x3.28.

#Sample Captures
This repo includes some captures including one of a repeat volume message to help figure out the CRC algorythm used.