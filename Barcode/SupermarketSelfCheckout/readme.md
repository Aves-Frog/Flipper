# Supermarket Self Checkout
> [!CAUTION]
> Tempering with self checkout systems or otherwise interfering with standard and intended operation of security devices is illegal.
> I do not condone tempering with systems you do not have explicit permissions for.

## How the barcode to exit system works
Note: This may differ depending on your region, store, or other implemented (security) features.

The barcodes I have encountered thus far work on the GS1-128 or EAN-128 codex, which allows for upto 48 characters of data within the barcode [^1]. 
The receipts I found used barcodes ranging from 14 to 20 digits [^2].
Which can be split up in different chunks of data.
Typically these chunks are a variation of the following: first digit indicating the receipt can be used, the code of the shop, the terminal/counter, the transaction number, the date.
Note that the usage or position of these chunks seems to be flexible.

As such a barcode such as 61234001999903072025 ([Receipt digit: 6] [Shop code: 1234] [Terminal: 001] [Transaction: 9999] [Date MMDDYYYY: 03072025]) would be a valid code.
Note that the receipts I used to decode these barcodes had clear indicators if not literal labels on it describing what each chunk meant.

From my findings I feel it is within reason to assume the exit gate scanunit checks the receipts on most chunks to be possible, this could easily be done in a lightweight lookup table.
But in order to speed up the process and limit bandwidth ignore some chunks, such as the transaction number (which btw simply loops and does not reset daily).
One of the leading reasons for this conclusion is the differences in complexity and thus cost comparing a database integrated system with a simpler solution, see below.

## A simple gate scaning solution
While there are probably as many simple solutions to this scanning unit as there are roads to Rome, I will outline two solutions I feel would be the most logical option for a technician to implement.
From a pure simplicity (and thus cost saving) standpoint I would say a Hardcoded solution or a Semi dynamic solution would be the most plausible choice.

### Requirements
For both the Hardcoded and Semi dynamic solution a simple chipboard (or even docker VM running on an admin pc) would provide more than enough power to run smoothly.
Both solutions would have file sizes in the bytes, as most data could be simplified.
For software both would need to run a simple script comparing int32 integer chunks next to a scan unit (these might even have a decoding chip build in, meaning the processing/checkings unit would only need int32 capabilities).
It really doesn't get much simpler in software.

### Functionality
Given the Hardcoded and Semi dynamic solutions function slightly differently I will explain them individually.

**Hardcoded solution:**

Within the code that checks the validity of the barcode we hardcode (define them within the code, to avoid needing to look them up) as many chunks as possible.
It stands to reason that the first digit, the static Receipt digit, doesn't need to change, so this could be hardcoded.
Next to that we can also assume that units get placed in the shop by a technician who has access to the unit's software, as such we can hardcode the shop code as well.
Other chunks that needn't change can be stored as well.

We now have (as for the previous example): 6-1234-XXX-XXXX-XXXXXXXXX

Now the Terminal number is the main argument that differs the Hardcoded solution from the Semi dynamic solution.
Which also for the Hardcoded solution has a few options.
The most logical option would be to have one variable (int32) reserved for storing the amount of terminals (and thus the valid values).
Example: for 5 counters -> static var 4 (keep in mind that integers start at 0).
Now actually starting the var at 1, i.e. flagging number 0 (or 000 as found in actual barcodes) as fraudulent would be a valid method to check this chunks validity aside from values above specified.

One downside of this method is that counters that are taken out of service cannot be flagged as invalid without overhauling the software.
Obviously this could be solved by adding a second code snipped that specifies and checks out of service counters (that are manually coded in).

Just for fun we could theorize other options that stay hardcoded. Such we could make a script that specifically defines each counter option. 
Just for fun and for the sake of ruining code efficiency we can even convert this into an if-statement list. 
Or even worse, just so the poor lonely CPU Hertz have something to do, create a class function that seeks to define the counter number and links to hardcoded class functions that all have the same function with their respective number.
This thought experiment is going off the rails, but it may show how optimized a Hardcoded solution may.

From here we have: 6-1234-001-XXXX-XXXXXXXX

Lastly assuming the scan unit has a time crystal (as do most motherboards) the time can be gathered locally, else using network connectivity the current data can be gathered.
Fun fact, even if no network connection or crystal is present the unit would still be able to note the date by simply counting upwards from the point of installation.
This is because the unit doesn't need to be precise in the hours. In theory a unit with a daily timing error or 2 minutes per day could be service free for 270 days. 

<sub>Assuming the unit is installed at midnight and the shop opens at 9am: 2minutes over 9 hours (to assume not one second of error during the open hours), 9hrs/2min -> 540/2 = 270. I.e. nearly one year </sub>

This gives us: 6-1234-001-XXXX-03072025

The only thing missing is the transaction number, but we can simply ignore this chunk. If you would really like to you could define specific numbers to check for errors such as 0000, 9999, 1234, etc.
But realistically there isn't much point.
As noted before these numbers simply loop whenever the highest number fitting (9999 or other defined) is hit at the counter.
This is where the Database solution shines, with it's database connection it can check in reasonable time whether this specific transaction is valid, or even if this transaction has already exited the gate.

**Semi dynamic solution:**

If we want to have a little more control on the fly we can use a lookup file (stored locally) such as an XML or JSON file.
Using such a file we can define the values of the receipt identifier (6) and the shop code in the lookup file, albeit with little realistic benefit.

More benefit can be found using this file to define the counters. This can simply be done by an integer such as in the Hardcoded option, or we can opt to define valid values.
Hence allowing technicians with less needed expertise (and lets face it less risk of bricking the device by not coding directly into the heart of the software) to quickly take Terminal numbers out of operation.
Another benefit is that you can change this lookup file (for example by USB connection or telnet) on the fly.

We could even build a platform that allows managers of the shop to take out counters from their pc. Or even better, have the counters themselves mark themselves as out of service (with a delay to allow checked out customers to exit).

As with the counters so too could the date be changed automatically by software on a pc.

I feel this Semi dynamic option would be a good sales pitch, 'empower your people and machines, reduce technician costs', but in all honestly not much more than a gimick.
Especially since this option still does not allow for transaction number checking as it isn't connected to a database.
Granted, this could be implemented, but that removes the benefit of not being plugged into a database natively, while introducing more delays.

### Limitations
While being a fast, cheap, and pretty low skill solution there are still some big limitations:
- This solution requires software technicians to operate locally for any changes or bugs
  - Alternatively a sofware package could be provided to make smaller changes by untrained personel, which still requires local fixing
- Units need to be reprogrammed (via change in lookup file value or literal code changes) to change counters, shop id, or time when desynced
- The biggest limitation; there is no (feasable) way to check transaction numbers, leaving a security flaw

## A database integrated solution
To mitigate (some of) the limitations mentioned above we can implement a Database solution. This would allow data to be stored in the cloud, and as such be changed externally by dedicated personel.
The biggest advantage of this solution is the ability to check the validity of transaction codes and patching this security hole.

### Requirements
This solution requires a bit more than the Hardcoded solution though:
- We need a unit that is capable of negotiating with the database (this means more CPU power and more RAM to store data locally)
- A managed server with the database, this should already be present to handle returns
- More skilled technicians who are familliar with software and database scripts such as SQL
- Bandwidth for the scan units
- Possibly dedicated network channels for these units (depending on how many there are and the bandwidth requirements)
- Local storage, while most operations should go through RAM we need a place to offload data

### Functionality
Depending on customer and technician preferences a databse based scanning unit would compare most of the data from the barcode against the database.
This could be done locally or by sending the barcode to a dedicated server, the process would be nearly similar in both cases. I would design the Database solution as such:

First check the recipt id number, this shouldn't ever need to change so can be hardcoded.
The shop code should also not need to change, however if really desired the unit could be saved in a different dataset and be assigned a shop code based on mac-address or similar.
This would allow a unit to be swapped and updated externally.
While I can see the use case, I feel this would not be worth the cost at all.

Based on the shop code a check could be made to the database to see what counters are marked as operative, from there the unit could check the Terminal value.
And the date could be grabbed from this same database.
Now while this sounds quite niche, this could actually be worth the bandwidth in some usecases to ensure all scanning units, pc's, and counters use the same date, and in case of a blackout or other event prevent the need of technicians to manually resync.

From the combination of the shop id, terminal and date a lookup request could be made to see if the transaction id is valid.

### Limitations
While mitigating some of the previous limitations we still have limitations with this solution:
- The biggest one being cost, technicians, bandwith, and hardware costs will be significantly higher
- The time to check the validity of the receipt will significantly increase
- In case of network loss or latency a transaction could be delayed or lost, causing the gate not to open

## Security exploits
Continueing to assume we operate using a non database connected solution we have a couple of security exploits that could happen, below I mentioned the ones I feel are most prevelant.
Note that I am only mentioning exploits that could happen to the exit scan unit, without destruction.

### Transaction fudging
Since we have established the scan units do not check for transaction id we can simply enter a random number in a fabricated barcode, while keeping the checked chunks untouched.

The biggest issue with this is that this exploit is pretty obvious visually.
Any security guard or personel standing around or watching CCTV will notice a random device being used instead of a receipt.
This could in theory be mitigated by using a receipt printer. This btw is the reason receipts have a custom backing depending on the store, it's just one more way to authenticate the receipt.
This issue is harder to spot in stores which allow members to exit using their loyalty cards as exit codes.
These barcodes are commonly stored on a mobile device such as a phone card manager, and therefore would not raise alarms when a phone with a phoney barcode would exit.
This ofcourse is best mitigated by using a database solution.

### Unit overloading
As anyone who has accidentally tried to exit using the wrong or crumbled up receipt can tell, you will still get a scan (or if you walked passed these scanners wearing a high vis jacket very many (if you know you know)).
This means that we could overload the unit by sending a lot of data into the scanner or into the script.

The scanner could possibly be overloaded by sending constant data. One way this could be done is by using reflective tape. The reflection makes most scanners go wild trying to decode the constantly changing paterns.
But given the low power usage of scanning this exploit isn't very practical. Besides, most scanners will return null when they cannot define the barcode type, as such this exploit wouldn't work.

At the script side a large amount of codes that conform to the standard could overload the software causing a failure (especially on network connected units such as database connected units as they have more latency and a bandwidth restriction). Now building safety code (in most countries) requires security measures and doors to open in case of failure, fire hazard, or power outage. Assuming the local code includes checkout gates to these measures overloading the unit could cause it to fall into fire open mode.
Probably the easiest countermeasure to this exploit is one that I see used in nearly all shops I have been to, namely to have a delay between codes scanned. 
If you need to wait say 3 seconds between barcode scan it not only allows honest users to adjust their barcode if it got read incorrectly, but also disallows people to overload the software via barcodes.

Another mitigation to this exploit is to create an audible clue. Some scan units I have encountered play a beep sound when they attempt a scan, a constant string of beeps would alert standers by and security personel.

[^1]: https://www.gs1us.org/upcs-barcodes-prefixes/gs1-128
[^2]: Plus and Jumbo supermarkets as found here (https://github.com/Aves-Frog/Flipper/new/main/Barcode/SupermarketSelfCheckout) 
