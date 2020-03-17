# Integrating physical devices with IOTA — Using RFID with IOTA
The 5th part in a series of beginner tutorials on integrating physical devices with the IOTA protocol.

![img](https://miro.medium.com/max/700/1*cU46NxoQ1z9fzQwopgsGIQ.jpeg)

------

## Introduction

This is the 5th part in a series of beginner tutorials where we explore integrating physical devices with the IOTA protocol. In this tutorial we will be using a Radio-Frequency Identification (RFID) reader/writer to capture information from an RFID tag that will be stored together with some other metadata on the IOTA tangle. We will also be creating a simple report that retrieves the same information from the tangle and displays it on a monitor.

This tutorial introduces a new topic that we will explore in the coming tutorials, namely the concept known as *Identity of Things*.

*Note!*
*Before we continue i would like to show my gratitude for all donations i have received. I’m blown away by the generosity and i want to ensure you all that it helps me stay motivated and focused. If this series can help anyone (especially kids) start playing around with electronics, programming and IOTA, my main motivation for writing these tutorials have been accomplished.*

------

## The Use Case

Now that our trusted hotel owner has his new refrigerator payment system up and running he wants to shift his attention to a different issue that has been causing him some headache. Today he uses a paper based system for tracking when and by whom each room in his hotel was cleaned and prepared for the next guest. The main problem with his paper based system is that there is a delay between the time when a room was cleaned to the time it can be made available for the next guest, depending on when the cleaning records was handed in to the receptionist. Another important aspect of these records is that they are used to identify the individual who performed the cleaning task in case of any dispute with respect to theft or the job not being properly done. These records are also used when paying his cleaning staff as each individual is payed by the number of rooms they clean each month. For this reason it is critical that the cleaning records are signed by the correct person and that they can’t be changed or tampered with later on. In addition, the fact that our hotel owner manages multiple hotels with a shared cleaning staff just adds to the complexity and makes all of this a logistical nightmare.

The first problem he faces when trying to replace his paper based system with a digital system is that there is no central network or database in place for all of his hotels. Each hotel operates individually while his cleaning staff rotates between them. The second problem is how to get the required information quickly and easily into the new system, replacing the hand written signature with a digital signature. The cleaning staff are not computer techies so it has to be simple to operate. The third problem he needs to figure out is how the receptionist can retrieve the cleaning records whenever he or she wants to check which rooms have been cleaned and made available for a new guest.

Now lets see if we can’t help him find a solution to these problems. Lets start with the first problem.

What if there was a permission less, tamper proof global database that could be used by anyone for free without the hassle of maintaining servers, network infrastructure, databases and user logins? Sounds to good to be true right? Fact is, there is such a database, and its called *The Tangle*.

The second problem we are going to solve using an RFID reader hooked up to an internet connected Raspberry PI placed at each floor of the hotel. Each person on the cleaning staff gets a unique ID card (RFID tag) that is used whenever the person creates a new record in the database. Replacing the manual signature with a digital signature.

The third problem we are going to solve by creating a simple report that can be run by the receptionist whenever he or she wants to check which rooms have been cleaned. The report retrieves the cleaning records from the tangle and displays it on a monitor in the reception.

So, lets get started…

------

## The MFRC522

The MFRC522 is a low-cost RFID reader/writer module that operates at 13.56 Mhz. The MFRC522 has gained a lot of popularity within the tinker community for its ease of use and Arduino/Raspberry PI support. You can get the MFRC522 module off ebay or Amazon for a couple of bucks. The RFID tags are very cheap (especially if you purchase them in larger bundles) and you can get them in the shape of a key ring or a credit card.

![img](https://miro.medium.com/max/600/1*6HQlyeYDEoNP_aaXOP-RiA.png)

## About Radio Frequency Identification (RFID)

Before we start wiring up and programming our MFRC522 module we should take a step back and have a look at the technology behind RFID and how it works. RFID is used in i variety of industries and chances are that you use it yourself on a daily basis without even thinking about it. Such as when you open a door in your office building with your employee ID card. RFID uses an electromagnetic field created by the RFID reader/writer for detecting and communicating with the tag. Inside the tag there is a small antenna in the form of a wire, and a small chip that carries identification and data about the tag. The tag does not need a battery as it is powered by the electromagnetic field generated by the reader/writer when held close by. The cool thing about these RFID tags is that, not only can you read the unique ID that was hard-coded into the tag when it was manufactured, you can also store your own data on to the RFID chip using the MFRC522 module. In this tutorial we will focus on reading and using the unique ID of the tag, but hopefully in a future tutorial we will be trying to write data to the tag as well. Stay tuned.

------

## Setting up the MFRC522

I found this [excellent tutorial](https://pimylifeup.com/raspberry-pi-rfid-rc522/) on connecting and setting up your MFRC522 with the Raspberry PI and i see no point in repeating the same steps in this tutorial. This is exactly what we need to get our MFRC522 up and running, so a big thanks to Gus for writing the tutorial.

*Note!
I think the wiring sketch in his* [*tutorial*](https://pimylifeup.com/raspberry-pi-rfid-rc522/) *have a minor error as the sketch points to pin 5 on the Raspberry PI for ground(GND). The correct pin should be pin 6 as noted in the list above the sketch.*

![img](https://miro.medium.com/max/700/1*vd3aw8dIUgJlMeuuSPt2HA.png)

------

## Required Software and libraries

Before we can start writing our Python code for this project we need to make sure that we have all the required software and libraries installed on our Raspberry PI.

Following the Gus’s “[How to setup a Raspberry Pi RFID RC522 Chip](https://pimylifeup.com/raspberry-pi-rfid-rc522/)” tutorial should take care of installing all the software and libraries required to the get the MFRC522 up and running with Python. The only additional libraries we now need to install is the [PyOTA library](https://github.com/iotaledger/iota.lib.py) (if you haven’t installed it already) for communicating with the IOTA tangle and the [PrettyTable](https://pypi.org/project/PrettyTable/) library used by our cleaning log report.

------

## The Python Code

Our Python code for this project will consist of two parts. One part is where we capture information from the cleaning staff and sends it to the tangle in the form of a new transaction. The second part is where we retrieve the same information from the tangle in the form of a cleaning log report that can be run by the receptionist. The information we are going to capture on the tangle consist of the following four elements:

1. The tag ID of the person who cleaned the room.
2. The name of the hotel.
3. The room number that was cleaned.
4. The date/time when the room was cleaned.

For the date/time element we will be using the transaction timestamp that was created when the transaction was published to the tangle.

We will be storing the information on the tangle in a json data format using the message element of the transaction. Using a structured json data format makes it convenient when we want to retrieve the same information for our cleaning log report later on.

The following Python code is the part where we capture the tagID from the RFID reader, together with the hotel name and room number, and sends it to the tangle.

```python
# Import datetime library
from datetime import datetime

# Import GPIO library
import RPi.GPIO as GPIO

# Import simplified version of the MFRC522 library
import SimpleMFRC522

# Import the PyOTA library
import iota

# Import json
import json

# Define IOTA address where all transactions (cleaning records) are stored, replace with your own address.
# IOTA addresses can be created with the IOTA Wallet
CleaningLogAddr = b"TAAUUAKUZHKBWTCGKXTJGWHXEYJONN9MZZQUQLSZCLHFAWUWKHZCICTHISXBBAKGFQENMWMBOVWJTCMEWXKQDTJCV9"

# Create IOTA object, specify full node to be used when sending transactions.
# Notice that not all nodes in the field.deviota.com cluster has enabled attaching transactions to the tangle
# In this case you will get an error, you can try again later or change to a different full node.
api = iota.Iota("https://nodes.thetangle.org:443")

# Define static variable
hotel = "Hotel IOTA View"

# Create RFID reader object
reader = SimpleMFRC522.SimpleMFRC522()

# Main loop, executes when an RFID tag (ID card) is close to the reader
try:
    while True:
        
        # Show welcome message
        print("\nWelcome to the Hotel IOTA cleaning log system")
        print("Press Ctrl+C to exit the system")
        
        # Get room number
        room_number = input("\nPlease type room number and press Enter: ")
        
        print("\nThank you, now hold your ID card near the reader")       
        
        # Get card ID from the reader
        id, text = reader.read()
                
        # Create json data to be uploaded to the tangle
        data = {'tagID': str(id), 'hotel': hotel, 'room_number': room_number}
        
        # Define new IOTA transaction
        pt = iota.ProposedTransaction(address = iota.Address(CleaningLogAddr),
                                      message = iota.TryteString.from_unicode(json.dumps(data)),
                                      tag     = iota.Tag(b'HOTELIOTA'),
                                      value   = 0)

        # Print waiting message
        print("\nID card detected...Sending transaction...Please wait...")

        # Send transaction to the tangle
        FinalBundle = api.send_transfer(depth=3, transfers=[pt], min_weight_magnitude=14)['bundle']
    
        # Print confirmation message 
        print("\nTransaction sucessfully completed, have a nice day")
                
        
# Clean up function when user press Ctrl+C (exit)
except KeyboardInterrupt:
    print("cleaning up")
    GPIO.cleanup()
```

You can download the source code from [here](https://gist.github.com/huggre/9f8fa49a6299b26ba1c49eddc3cafa04)

The following Python code is used by the receptionist when retrieving and displaying the cleaning log.

```python
# Imports from the PyOTA library
from iota import Iota
from iota import Address
from iota import Transaction
from iota import TryteString

# Import json library
import json

# Import datetime libary
import datetime

# Import from PrettyTable
from prettytable import PrettyTable

# Define IOTA address where all transactions (cleaning records) are stored, replace with your own address.
address = [Address(b'TAAUUAKUZHKBWTCGKXTJGWHXEYJONN9MZZQUQLSZCLHFAWUWKHZCICTHISXBBAKGFQENMWMBOVWJTCMEWXKQDTJCV9')]

# Define full node to be used when retrieving cleaning records
iotaNode = "https://nodes.thetangle.org:443"

# Create an IOTA object
api = Iota(iotaNode)

# Create PrettyTable object
x = PrettyTable()

# Specify column headers for the table
x.field_names = ["tagID", "Hotel", "Room", "last_cleaned"]

# Find all transacions for selected IOTA address
result = api.find_transactions(addresses=address)
    
# Create a list of transaction hashes
myhashes = result['hashes']

# Print wait message
print("Please wait while retrieving cleaning records from the tangle...")

# Loop trough all transaction hashes
for txn_hash in myhashes:
    
    # Convert to bytes
    txn_hash_as_bytes = bytes(txn_hash)

    # Get the raw transaction data (trytes) of transaction
    gt_result = api.get_trytes([txn_hash_as_bytes])
    
    # Convert to string
    trytes = str(gt_result['trytes'][0])
    
    # Get transaction object
    txn = Transaction.from_tryte_string(trytes)
    
    # Get transaction timestamp
    timestamp = txn.timestamp
    
    # Convert timestamp to datetime
    clean_time = datetime.datetime.fromtimestamp(timestamp).strftime('%Y-%m-%d %H:%M:%S')

    # Get transaction message as string
    txn_data = str(txn.signature_message_fragment.decode())
    
    # Convert to json
    json_data = json.loads(txn_data)
    
    # Check if json data has the expected json tag's
    if all(key in json.dumps(json_data) for key in ["tagID","hotel","room_number"]):
        # Add table row with json values
        x.add_row([json_data['tagID'], json_data['hotel'], json_data['room_number'], clean_time])

# Sort table by cleaned datetime
x.sortby = "last_cleaned"

# Print table to terminal
print(x)
```

You can download the source code from [here](https://gist.github.com/huggre/62fed3834637738d186ce2fecd35c66f)

When running the report you should get a result similar to this.

![img](https://miro.medium.com/max/545/1*1sYM2d-GeULqxGWgT92Ifw.png)



------

## Running the project

To run the first part of the project, you first need to save the code in the previous section as a text file in the same folder as where you installed the SimpleMFRC522 library.

Notice that Python program files uses the .py extension, so let’s save the file as **cleaning_register.py** on the Raspberry PI.

To execute the program, simply start a new terminal window, navigate to the folder where you saved *cleaning_register.py* and type:

**python cleaning_register.py**

You should now see the code being executed in your terminal window.

The program will start by displaying a welcome message and then ask for the room number that was cleaned. After the user enters the room number and press Enter, the program will ask the user to hold his/her ID card close to the RFID reader. As soon as RFID reader detects the ID card (RFID tag), it will read the unique ID stored on the card and send a new transaction to the tangle, including the additional metadata as a transaction message.

As soon as a new transaction has been sent to the tangle you should be able to find it using your favorite tangle browser, such as [thetangle.org](https://thetangle.org/)

The second part of the project can be run from any internet connected computer that has Python and the other required libraries installed, namely [PyOTA](https://github.com/iotaledger/iota.lib.py) and [PrettyTable](https://pypi.org/project/PrettyTable/).

Save the file on the computer as **cleaning_log.py** and execute the program by typing **python cleaning_log.py**

After executing the program you should see the cleaning log displayed in the terminal window, listing each individual cleaning record stored on the tangle.

*Note!*
*Make sure you use the same IOTA address in both Python scripts, otherwise you will not get any records displayed in the log.*

## Donations

If you like this tutorial and want me to continue making others, feel free to make a small donation to the IOTA address shown below.

![img](https://miro.medium.com/max/382/1*j2ENIzmDzXcGSgAdY4w-Jw.png)

NYZBHOVSMDWWABXSACAJTTWJOQRPVVAWLBSFQVSJSWWBJJLLSQKNZFC9XCRPQSVFQZPBJCJRANNPVMMEZQJRQSVVGZ

