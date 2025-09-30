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
    Duration = Table.AddColumn(StartDate, "DurationMinutes", each 
        if [endTime] <> null then Number.Round(([endTime]-[startTime]) / 60000, 1) else null, type nullable number),

    // Keep only the columns you want
    Final = Table.SelectColumns(Duration, {"id","title","Impacted","Affected","RootCause","StartDate","DurationMinutes"})
in
    Final
