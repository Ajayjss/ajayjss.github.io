Made by ajayjss
let
    // Load JSON file (adjust file path if needed)
    Source = Json.Document(File.Contents("C:\Users\V1018598.CORP\Downloads\response_1759219549680.json")),

    // Drill into the "problems" array
    problems = Source[problems],

    // Turn the list into a table
    problemsTable = Table.FromList(problems, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    Expanded = Table.ExpandRecordColumn(
        problemsTable,
        "Column1",
        {"id","title","startTime","endTime","impactAnalysis","evidenceDetails"},
        {"id","title","startTime","endTime","impactAnalysis","evidenceDetails"}
    ),

    // Impacted Entities
    Impacted = Table.AddColumn(Expanded, "Impacted", each Text.Combine(
        List.Transform(
            Record.FieldOrDefault([impactAnalysis],"impactedEntities",{}),
            each Record.FieldOrDefault(_,"name","")
        ), ", "), type text),

    // Affected Entities
    Affected = Table.AddColumn(Impacted, "Affected", each Text.Combine(
        List.Transform(
            Record.FieldOrDefault([impactAnalysis],"affectedEntities",{}),
            each Record.FieldOrDefault(_,"name","")
        ), ", "), type text),

    // Root Cause
    RootCause = Table.AddColumn(Affected, "RootCause", each 
        let
            ed = if [evidenceDetails] = null then {} else [evidenceDetails],
            rcs = List.Select(ed, each Record.FieldOrDefault(_,"rootCauseRelevant", false) = true),
            details = List.Transform(rcs, each Record.FieldOrDefault(_,"details",""))
        in
            Text.Combine(details, "; "), type text),

    // Start Date (convert epoch ms to datetime)
    StartDate = Table.AddColumn(RootCause, "StartDate", each 
        #datetime(1970,1,1,0,0,0) + ([startTime]/1000)/86400, type datetime),

    // Duration in minutes
let
    // Load JSON file (adjust path or replace with Web.Contents if calling API)
    Source = Json.Document(File.Contents("C:\Users\YourName\Downloads\response.json")),

    // Drill into problems array
    Problems = Source[problems],

    // Convert to table
    ProblemsTable = Table.FromList(Problems, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    ExpandedProblems = Table.ExpandRecordColumn(
        ProblemsTable,
        "Column1",
        {"problemId", "displayId", "title", "status", "startTime", "endTime", "impactedEntities", "managementZones"}
    ),

    // Expand impactedEntities
    ExpandImpactedEntities = Table.ExpandListColumn(ExpandedProblems, "impactedEntities"),
    ExpandImpactedEntityDetails = Table.ExpandRecordColumn(
        ExpandImpactedEntities,
        "impactedEntities",
        {"name"},
        {"impactedEntityName"}
    ),

    // Expand managementZones
    ExpandZones = Table.ExpandListColumn(ExpandImpactedEntityDetails, "managementZones"),
    ExpandZoneDetails = Table.ExpandRecordColumn(
        ExpandZones,
        "managementZones",
        {"name"},
        {"managementZoneName"}
    ),

    // Convert epoch to datetime (start & end)
    AddStartDate = Table.TransformColumns(
        ExpandZoneDetails,
        {{"startTime", each #datetime(1970,1,1,0,0,0) + #duration(0,0,0, Number.Round(_ / 1000)), type datetime}}
    ),
    AddEndDate = Table.TransformColumns(
        AddStartDate,
        {{"endTime", each #datetime(1970,1,1,0,0,0) + #duration(0,0,0, Number.Round(_ / 1000)), type datetime}}
    ),

    // Calculate duration in minutes
    AddDurationMinutes = Table.AddColumn(AddEndDate, "DurationMinutes", each Duration.TotalMinutes([endTime] - [startTime]), type number)

in
    AddDurationMinutes
