---
author: "Kristen Tidmuss"
title: "EvE Connection Finder"
date: "2020-02-11"
description: "An EvE Online tool to find characters with similar corp history"
tags: ["EvE Online", "C#", "MVC"]
draft: false
hideMeta: false
ShowToc: true
cover:
    image: "<img>" # image path/url
    alt: "<alt text>" # alt text
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
[![GitHub](/images/github.png)](https://github.com/KristenTidmuss/EveConnectionFinder)

# Intro
This is a small project I created along side my friend Clayton. I came up with the idea when more and more frequently I would login to the game EvE Online after long breaks and have people recognise me and I would have no idea who they were.

To solve this issue I have created EvE Connection Finder, Simply type your name into the top box and either paste an individual or the entirety of the system into the second and hit submit. This will then take between a few seconds and a minute to process. Once complete a list of anyone you share mutual history with will be shown below.
![Webpage Demo](/images/projects/EveConnectionFinder.png)

# Code
## Architecture
This project is made using the new .NET 5 with ASP.net MVC as the backend. When the user submits a request the server then goes and gets all the valid information from EvEs ESI and compares it all to generate a list of common corporations.
## Main Page
The main page has very little code itself. Once you have filled in the main character name and a list to compare with a simle process begins.

As you can see here firstly it get the Character ID and corp history for the main character then proceeds to loop through every user pasted. This can take some time and in testing a large paste of 40 or so users could take up to 15 minutes due to the speed of ESI and rate of errors it throws. To solved this I implemented Parallels so multiple users can be processed at once.
```C#
//Data is returned from the form here
//Process Details for user character
var userCharacter = new Character {charName = form.UserCharacter};
userCharacter.GetCharID();
userCharacter.GetCorps();
//Process details for paste
var pastedCharacters = new List<Character>();
string[] charList =
    form.PasteList.Replace(userCharacter.charName,"").Split(new char[] {'\r', '\n'}, StringSplitOptions.RemoveEmptyEntries);
//Loop each user to get their corp history
Parallel.ForEach(charList, name =>
{
    var character = new Character {charName = name};
    character.GetCharID();
    character.GetCorps();
    pastedCharacters.Add(character);
});
//Find connections
ViewBag.connections = userCharacter.FindConnections(pastedCharacters)
    .OrderByDescending(c => c.overlapStart);
```
## Processing Corps
The process for getting all the corp history is also rather simple once you get your head around the main concepts, the main difficulty was finding the end dates of corp membership however a small foreach loop solved that.

As you can see I chose to store the data in a List object which I find just make it easier to work with. Then all I needed to do was call for he corp history then loop through each entry to get the full details of the corp.
```C#
public void GetCorps()
{    //Get history list
    string corpHistoryJson = DownloadString("https://esi.evetech.net/latest/characters/" + this.charID + "/corporationhistory datasource=tranquility");
    var corpHistoryResult = JsonConvert.DeserializeObject<List<ESICorpHistory>>(corpHistoryJson);
    //Process into Corp class
    this.Corps = new List<Corp>();
    Parallel.ForEach(corpHistoryResult, history =>
    {
        //corp name
        string corpNameSearch = DownloadString("https://esi.evetech.net/latest/corporations/" + history.corporation_id + "/?datasource=tranquility");
        var corpNameSearchResult = JsonConvert.DeserializeObject<ESICorp>(corpNameSearch);
        //
        var corp = new Corp
        {
            corpID = history.corporation_id,
            corpName = corpNameSearchResult.name,
            startDate = DateTime.Parse(history.start_date)
        };
        Corps.Add(corp);
    });
    //Populate end dates
    Corp previousCorp = null;
    foreach (var c in Corps.OrderByDescending(c => c.startDate))
    {
        if (previousCorp != null)
        {
            c.endDate = previousCorp.startDate;
        }
        else
        {
            c.endDate = DateTime.MaxValue;
        }
        previousCorp = c;
    }
}
```
## Finding Connections
The last step was to find the connections. This was rather difficult to figure out however I believe I found an efficient way to do so and although it may look a bit bulky it takes no time at all to actually process.

The way this works is first, excluding all rookie NPC corps as they are irrelevant to the point of this task. Next it runs foreach of the main characters corp and another foreach of users matching their history. Then for each of those characters it gets the overlapping corp and calculates the start/end dates where you shared the history.
```C#
public List<CharacterConnection> FindConnections(List<Character> pastedCharacters)
{
    var Connections = new List<CharacterConnection>();
    //Define rookie corps to ignore
    var rookieCorps = new string[] { "Hedion University", "Imperial Academy", "Royal Amarr Institute", "School of Applied Knowledge", "Science and Trade Institute", "State War Academy", "Center for Advanced Studies", "Federal Navy Academy", "University of Caille", "Pator Tech School", "Republic Military School", "Republic University", "Viziam", "Ministry of War", "Imperial Shipment", "Perkone", "Caldari Provisions", "Deep Core Mining Inc.", "The Scope", "Aliastra", "Garoun Investment Bank", "Brutor Tribe", "Sebiestor Tribe", "Native Freshfood", "Ministry of Internal Order", "Expert Distribution", "Impetus", "Brutor Tribe", "Amarr Imperial Navy", "Ytiri", "Federal Intelligence Office", "Republic Security Services", "Dominations", "Guristas"};
    //Loop Chars corp history
    foreach (var corp in this.Corps.Where(c => !rookieCorps.Contains(c.corpName)))
    {
        //get pasted chars who share that corp at same time
        foreach (var match in pastedCharacters.Where(p => p.Corps.Count(c => c.corpID == corp.corpID && c.startDate < corp.endDate && corp.startDate < c.endDate) > 0))
        {
            var matchEntry = match.Corps.FirstOrDefault(c => c.corpID == corp.corpID && c.startDate < corp.endDate && corp.startDate < c.endDate);
            //Calculate overlap dates
            DateTime entityStart;
            DateTime entityEnd;
            if(matchEntry.startDate > corp.startDate) { entityStart = matchEntry.startDate;} else { entityStart = corp.startDate;}
            if(matchEntry.endDate > corp.endDate) { entityEnd = corp.endDate;} else { entityEnd = matchEntry.endDate;}
            var connection = new CharacterConnection()
            {
                charID = match.charID,
                charName = match.charName,
                entityID = corp.corpID,
                entityName = corp.corpName,
                overlapStart = entityStart,
                overlapEnd = entityEnd
            };
            Connections.Add(connection);
        }
    }
    return Connections;
}
```
## Alliances
Alliances are a whole different beast and I am yet to dream up an efficient way to calculate those. I know it will involve pulling the alliance history for every corp in every players history but then building a timeline and matching it all up is way over my head. This is an ongoing project though and I am confident I will figure it out some day!