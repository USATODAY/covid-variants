
# DEFUNCT PROJECT
## The CDC killed this data source
In mid-April 2021, the Centers for Disease Control killed off the data source. The one they're referring us to doesn't have easily downloadable data and isn't updated nearly as frequently. Details: https://www.cdc.gov/coronavirus/2019-ncov/transmission/variant-cases.html


# Coronavirus variant data

This repository of state-level COVID-19 / coronavirus variants  contains data originally sourced from the Centers for Disease Control and Prevention. A few of the earlier files were pulled through the Wayback Machine. A few of the earlier files were altered to make them easier to parse.

If everything works and the CDC has not broken the parser, this data should update within 30 minutes of a CDC change.

**!combined.csv** and **combined.json** should be automatically updated if the CDC hasn't broken the USA TODAY parser implementation. You may be able to build around Github-hosted files using the "raw" view such as https://raw.githubusercontent.com/USATODAY/covid-variants/master/combined.json ... 

Per MIT license, no warranty is implied.

The CDC's release schedule has changed several times; the promised time of updates has often been ignored. The CDC's source page for this data is available at:
https://www.cdc.gov/coronavirus/2019-ncov/transmission/variant-cases.html

The data has been appearing most recently at:
https://www.cdc.gov/coronavirus/2019-ncov/modules/transmission/variant-cases.json

Note there have been at least three URLs for this data. The data is getting more stable, but it's problematic.

Some sample code for parsing the JSON:

    import simplejson as json
    
    from glob import glob
    import os
    from collections import OrderedDict
    
    basedir = ""
    csvdir = ""
    jsondir = csvdir + "json/"
    for localdir in [csvdir, jsondir, miscdir]:
        os.makedirs(localdir, exist_ok=True)
    fileprefix = "cdc-variants-"
    goodvariants = ["B.1.1.7 Variant", "P.1 Variant", "B.1.351 Variant"]
    
    jsonfiles = sorted(list(glob(jsondir + "*.json")))
    
    
    def clean_entry(entry):
        line = OrderedDict()
        if len(entry) == 2:
            for item in entry:
                if item == "Cases":
                    line['B.1.1.7 Variant'] = entry['Cases']
                else:
                    line[item] = entry[item]
        else:
            for item in entry:
                if item not in ["filter", "Cases", "Range"]:
                    line[item.strip()] = entry[item]
        if line['State'] == "District of Columbia":    # Hacky standardization
            line['State'] = "DC"
        return(line)
    
    
    # First, get all the possible headers from these things
    headers = ["filedate"]
    for jsonfile in jsonfiles:
        with open(jsonfile, "r") as infile:
            localdata = json.load(infile)['data']
            for entry in localdata:
                row = clean_entry(entry)
                for item in row:
                    if item not in headers:
                        headers.append(item)
    headers.append("mytotal")
    print(f"Headers found: {' ... '.join(headers)}")
    
    # Now, start parsing
    masterdict = {}
    filedatelist = []
    for jsonfile in jsonfiles:
        filedate = jsonfile[-15:][:-5]
        filedatelist.append(filedate)
        with open(jsonfile, "r") as infile:
            localdata = json.load(infile)['data']
            for entry in localdata:
                line = OrderedDict()
                row = clean_entry(entry)
                state = row['State']
                if state not in [None, "Total", ""]:
                    for item in headers:
                        line[item] = None
                    line["filedate"] = filedate    # and we're done initializing the row, maybe
                    line['State'] = state
                    for goodvariant in goodvariants:
                        line[goodvariant] = 0
                    line['mytotal'] = 0
                    for item in row:
                        line[item] = row[item]
                        if item in goodvariants and isinstance(line[item], str):    # Force string to numbers
                            if len(line[item]) == 0:
                                line[item] = 0
                            else:
                                line[item] = int(line[item])
                        if item in goodvariants and not line[item]:   # If we're supplied with null -> None, drop in 0
                            line[item] = 0
                    for goodvariant in goodvariants:
                        localvalue = line[goodvariant]
                        if localvalue:
                            line["mytotal"] += localvalue
                    # Now we have a line that should be fairly complete. But we're no longer appending to a list of lines.
                    # Let's shoehorn into the new format.
                    if state not in masterdict:
                        masterdict[state] = OrderedDict()
                    if filedate not in masterdict[state]:
                        masterdict[state][filedate] = line
                    for goodvariant in goodvariants:
                        if line[goodvariant] > masterdict[state][filedate][goodvariant]:
                            masterdict[state][filedate][goodvariant] = line[goodvariant]
                
	    
    # Revised D.C. hack. We may be missing some of the earliest geo entries, so let's fake them all.
    for state in masterdict:
        localfails = []
        for filedate in filedatelist:
            if filedate not in masterdict[state]:
                localfails.append(filedate)
                line = OrderedDict()
                for item in headers:
                    line[item] = None
                line['State'] = state
                line['filedate'] = filedate
                for item in goodvariants:
                    line[item] = 0
                line['mytotal'] = 0
                masterdict[state][filedate] = line
        if len(localfails) > 0:
            print(f"Added backdated entries for {state}: {' ... '.join(localfails)}")
     	 
    
    for state in masterdict:
        for mydate in masterdict[state]:
            localtally = 0
            for goodvariant in goodvariants:
                localtally += masterdict[state][mydate][goodvariant]
            masterdict[state][mydate]["mytotal"] = localtally        
    
    # Third D.C. patch
    tempdict = OrderedDict()
    for state in sorted(masterdict):
        tempdict[state] = OrderedDict()
        for mydate in sorted(masterdict[state]):
            tempdict[state][mydate] = masterdict[state][mydate]
    masterdict = tempdict
    tempdict = None
    
    with open(f"{csvdir}!combined.csv", "w", newline="") as outfile:
        writer = csv.writer(outfile)
        writer.writerow(headers)
        for state in masterdict:
            for mydate in masterdict[state]:
                writer.writerow(list(masterdict[state][mydate].values()))


