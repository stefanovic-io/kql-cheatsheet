# search, where, take, count, summarize, extend, project, distinct

# Search

// Will search all columns in the Perf table for the value "Memory"

**Perf 
| search "Memory"**

// Although you can make it sensitive using a switch

**Perf 
| search kind=case_sensitive "memory" **

// A better choice is to limit the search to specific tables

**search in (Perf, Event, Alert) "Contoso"**

// The syntax......: Perf | search "Contoso"

// is the same as..: search in (Perf) "Contoso"

// Within a table, you can search a specific column for an exact value

**Perf 
| search CounterName=="Available MBytes"**


// Can also search for the value anywhere in the text in the specified column

**Perf 
| search CounterName:"MBytes"**


// This searched for a column whose value exactly matched the word Memory

**Perf 
| search "Memory"**


// Can also search across all columns using wildcards

**Perf 
| search "*Bytes*"**                  // Anywhere in any column's text

**Perf
| search * startswith "Bytes"**       // Begins with Bytes then any text after it

**Perf
| search * endswith "Bytes"**         // Ends with Bytes

**Perf 
| search "Free*bytes"** // Begins with Free, ends with bytes, anything in between


// Searches can be combined logically

**Perf 
| search "Free*bytes" and ("C:" or "D:")**


// Search also supports regular expressions

**Perf 
| search InstanceName matches regex "[A-Z]:"**


# Where

// Similar to search, where limits the result set. Rather than looking across
// columns for values, where limits based on conditions

**Perf 
| where TimeGenerated >= ago(1h)**


// ago allows for relative date ranges. Ago says "start right now, then go back

// in time N quantity. These can be expressed using the following abbreviations

//            d - days

//            h - hours

//            m - minutes

//            s - seconds

//           ms - milliseconds

//  microsecond - microseconds


// Can build up the where by adding to it logically

**Perf 
| where TimeGenerated  >= ago(1h)
    and CounterName == "Bytes Received/sec"**


// Multiple ands are allowed

**Perf 
| where TimeGenerated  >= ago(1h)
    and CounterName == "Bytes Received/sec"
    and CounterValue > 0****


// OR logic is allowed too!

**Perf 
| where TimeGenerated  >= ago(1h)
    and (CounterName == "Bytes Received/sec"
         or  
         CounterName == "% Processor Time"
        )**


// Combining and's and or's

**Perf 
| where TimeGenerated  >= ago(1h)
    and (CounterName == "Bytes Received/sec"
         or
         CounterName  == "% Processor Time"
        )
    and CounterValue > 0**


// Stackable where operators

**Perf 
| where TimeGenerated  >= ago(1h)
| where (CounterName == "Bytes Received/sec"
         or
         CounterName  == "% Processor Time"
        )
| where CounterValue > 0**


// You can simulate search using where. Here it searches all columns

// in the input for the phrase Bytes somewhere in a column's value

**Perf 
| where * has "Bytes"**


// Similar to search, you can search positional matches

**Perf
| where * hasprefix "Bytes" ** // At the start

**Perf
| where * hassuffix "Bytes" ** // At the end

**Perf
| where * contains "Bytes" **  // contains and has behave the same


// Where supports regex as well

**Perf
| where InstanceName matches regex "[A-Z]:"**



# Take


// Take is used to grab a random number of rows from the input data

**Perf 
| take 10 **


// There is no guarantee to which rows are returned, and the exact same

// query run a second time may result in a different set of rows

// (or it may be the same, depends on various factors)

**Perf 
| take 10 **


// Take can be combined with other language operators

**Perf 
| where TimeGenerated  >= ago(1h)
    and CounterName == "Bytes Received/sec"
    and CounterValue > 0
| take 5**


// Limit is a synonym for take
Perf 
| limit 10


//------------------------------------------------------------------------------
// Count
//------------------------------------------------------------------------------

// Returns the number of rows in the input dataset
Perf 
| count 


// Can also use with other filters
Perf 
| where TimeGenerated  >= ago(1h)
    and CounterName == "Bytes Received/sec"
    and CounterValue > 0
| count


//------------------------------------------------------------------------------
// Summarize
//------------------------------------------------------------------------------

// Summariaze allows you count number of rows by column using the count()
// aggregation function
Perf 
| summarize count() by CounterName


// Can break down by multiple columns
Perf 
| summarize count() by ObjectName, CounterName


// You can rename the output column for count
Perf 
| summarize PerfCount=count() 
         by ObjectName, CounterName


// With Summarize, you can  use other aggregation functions
Perf 
| where CounterName == "% Free Space"
| summarize NumberOfEntries=count()
          , AverageFreeSpace=avg(CounterValue) 
         by CounterName


// Bin allows you to summarize into logical groups, like days
Perf 
| summarize NumberOfEntries=count() 
         by bin(TimeGenerated, 1d)


// Can bin at multiple levels
Perf 
| summarize NumberOfEntries=count() 
         by CounterName
          , bin(TimeGenerated, 1d)


// Order in the query determines the order in the output
Perf 
| summarize NumberOfEntries=count() 
         by bin(TimeGenerated, 1d)
          , CounterName


// You can bin by values other than dates.
// Here we see number of entries for each level at 
// % Free Space
Perf
| where CounterName == "% Free Space"
| summarize NumberOfRowsAtThisPercentLevel=count() 
         by bin(CounterValue,10)


//------------------------------------------------------------------------------
// Extend
//------------------------------------------------------------------------------

// Extend creates a calculated column and adds to the result set
Perf 
| where CounterName == "Free Megabytes"
| extend FreeGB = CounterValue / 1000


// Can extend multiple columns at the same time
Perf 
| where CounterName == "Free Megabytes"
| extend FreeGB = CounterValue / 1000
       , FreeKB = CounterValue * 1000


// Could use to repeat a column
Perf 
| where CounterName == "Free Megabytes"
| extend FreeGB = CounterValue / 1000
       , FreeMB = CounterValue 
       , FreeKB = CounterValue * 1000


// Can also use with strcat to create new string columns
Perf
| extend ObjectCounter = strcat(ObjectName, " - ", CounterName) 


//------------------------------------------------------------------------------
// Project
//------------------------------------------------------------------------------

// Project allows you to select a subset of columns
Perf
| project ObjectName
        , CounterName 
        , InstanceName 
        , CounterValue 
        , TimeGenerated 


// Combine Project with Extend
Perf
| where CounterName == "Free Megabytes"
| project ObjectName
        , CounterName 
        , InstanceName 
        , CounterValue 
        , TimeGenerated 
| extend FreeGB = CounterValue / 1000
       , FreeMB = CounterValue 
       , FreeKB = CounterValue * 1000


// You can use extend prior to project to calculate on a field
// you don't want in the final output (Counter Value omitted)
Perf
| where CounterName == "Free Megabytes"
| extend FreeGB = CounterValue / 1000
       , FreeMB = CounterValue 
       , FreeKB = CounterValue * 1000
| project ObjectName
        , CounterName 
        , InstanceName 
        , TimeGenerated 
        , FreeGB 
        , FreeKB 
        , FreeMB 


// Project can simulate the extend
Perf
| where CounterName == "Free Megabytes"
| project ObjectName 
        , CounterName 
        , InstanceName 
        , TimeGenerated 
        , FreeGB = CounterValue / 1000
        , FreeMB = CounterValue 
        , FreeKB = CounterValue * 1000


// There is a variant called project-away. It will project all except the 
// columns listed
Perf
| where TimeGenerated > ago(1h)
| project-away TenantId
             , SourceSystem 
             , CounterPath 
             , MG


// If you only want to rename a column, then another variant of project is
// project-rename. It will rename the specified column but then pass the
// rest of the columns through
Perf
| where TimeGenerated > ago(1h)
| project-rename myRenamedComputer = Computer 


//------------------------------------------------------------------------------
// Distinct//------------------------------------------------------------------------------
// Kusto Query Language (KQL) From Scratch
// Module 2 - 80% of the Operators You'll Ever Use
//
// The demos in this module serve as a very basic introduction to the KQL
// language within the Azure Log Analytics environment. 
//
// Copyright (c) 2018. Microsoft, Pluralsight, Robert C. Cain. 
// All rights reserved. This code may be used in part within your own
// applications. 
//
// This code may NOT be redistributed in it's entirely without permission
// of one of it's copyright holders. 
//------------------------------------------------------------------------------

//------------------------------------------------------------------------------
// Search
//------------------------------------------------------------------------------

// Will search all columns in the Perf table for the value "Memory"
Perf 
| search "Memory"


// Search is not case sensitive by default
Perf 
| search "memory"


// Although you can make it sensitive using a switch
Perf 
| search kind=case_sensitive "memory" 


// Without a table, search goes arcross all tables in the current database
// Warning, this takes a looooooooooong time to run, and it is highly likely
// it will time out on a large database, like the Log Analytics Demo database. 
search "Memory"


// A better choice is to limit the search to specific tables
search in (Perf, Event, Alert) "Contoso"

// The syntax......: Perf | search "Contoso"
// is the same as..: search in (Perf) "Contoso"


// Within a table, you can search a specific column for an exact value
Perf 
| search CounterName=="Available MBytes"


// Can also search for the value anywhere in the text in the specified column
Perf 
| search CounterName:"MBytes"


// This searched for a column whose value exactly matched the word Memory
Perf 
| search "Memory"


// Can also search across all columns using wildcards
Perf 
| search "*Bytes*"                  // Anywhere in any column's text

Perf
| search * startswith "Bytes"       // Begins with Bytes then any text after it

Perf
| search * endswith "Bytes"         // Ends with Bytes

Perf 
| search "Free*bytes" // Begins with Free, ends with bytes, anything in between


// Searches can be combined logically
Perf 
| search "Free*bytes" and ("C:" or "D:")


// Search also supports regular expressions
Perf 
| search InstanceName matches regex "[A-Z]:"


//------------------------------------------------------------------------------
// Where
//------------------------------------------------------------------------------

// Similar to search, where limits the result set. Rather than looking across
// columns for values, where limits based on conditions
Perf 
| where TimeGenerated >= ago(1h)


// ago allows for relative date ranges. Ago says "start right now, then go back
// in time N quantity. These can be expressed using the following abbreviations
//            d - days
//            h - hours
//            m - minutes
//            s - seconds
//           ms - milliseconds
//  microsecond - microseconds


// Can build up the where by adding to it logically
Perf 
| where TimeGenerated  >= ago(1h)
    and CounterName == "Bytes Received/sec"


// Multiple ands are allowed
Perf 
| where TimeGenerated  >= ago(1h)
    and CounterName == "Bytes Received/sec"
    and CounterValue > 0


// OR logic is allowed too!
Perf 
| where TimeGenerated  >= ago(1h)
    and (CounterName == "Bytes Received/sec"
         or  
         CounterName == "% Processor Time"
        )


// Combining and's and or's
Perf 
| where TimeGenerated  >= ago(1h)
    and (CounterName == "Bytes Received/sec"
         or
         CounterName  == "% Processor Time"
        )
    and CounterValue > 0


// Stackable where operators
Perf 
| where TimeGenerated  >= ago(1h)
| where (CounterName == "Bytes Received/sec"
         or
         CounterName  == "% Processor Time"
        )
| where CounterValue > 0


// You can simulate search using where. Here it searches all columns
// in the input for the phrase Bytes somewhere in a column's value
Perf 
| where * has "Bytes"


// Similar to search, you can search positional matches
Perf
| where * hasprefix "Bytes"  // At the start

Perf
| where * hassuffix "Bytes"  // At the end

Perf
| where * contains "Bytes"   // contains and has behave the same


// Where supports regex as well
Perf
| where InstanceName matches regex "[A-Z]:"


//------------------------------------------------------------------------------
// Take
//------------------------------------------------------------------------------

// Take is used to grab a random number of rows from the input data
Perf 
| take 10 


// There is no guarantee to which rows are returned, and the exact same
// query run a second time may result in a different set of rows
// (or it may be the same, depends on various factors)
Perf 
| take 10 


// Take can be combined with other language operators
Perf 
| where TimeGenerated  >= ago(1h)
    and CounterName == "Bytes Received/sec"
    and CounterValue > 0
| take 5


// Limit is a synonym for take
Perf 
| limit 10


//------------------------------------------------------------------------------
// Count
//------------------------------------------------------------------------------

// Returns the number of rows in the input dataset
Perf 
| count 


// Can also use with other filters
Perf 
| where TimeGenerated  >= ago(1h)
    and CounterName == "Bytes Received/sec"
    and CounterValue > 0
| count


//------------------------------------------------------------------------------
// Summarize
//------------------------------------------------------------------------------

// Summariaze allows you count number of rows by column using the count()
// aggregation function
Perf 
| summarize count() by CounterName


// Can break down by multiple columns
Perf 
| summarize count() by ObjectName, CounterName


// You can rename the output column for count
Perf 
| summarize PerfCount=count() 
         by ObjectName, CounterName


// With Summarize, you can  use other aggregation functions
Perf 
| where CounterName == "% Free Space"
| summarize NumberOfEntries=count()
          , AverageFreeSpace=avg(CounterValue) 
         by CounterName


// Bin allows you to summarize into logical groups, like days
Perf 
| summarize NumberOfEntries=count() 
         by bin(TimeGenerated, 1d)


// Can bin at multiple levels
Perf 
| summarize NumberOfEntries=count() 
         by CounterName
          , bin(TimeGenerated, 1d)


// Order in the query determines the order in the output
Perf 
| summarize NumberOfEntries=count() 
         by bin(TimeGenerated, 1d)
          , CounterName


// You can bin by values other than dates.
// Here we see number of entries for each level at 
// % Free Space
Perf
| where CounterName == "% Free Space"
| summarize NumberOfRowsAtThisPercentLevel=count() 
         by bin(CounterValue,10)


//------------------------------------------------------------------------------
// Extend
//------------------------------------------------------------------------------

// Extend creates a calculated column and adds to the result set
Perf 
| where CounterName == "Free Megabytes"
| extend FreeGB = CounterValue / 1000


// Can extend multiple columns at the same time
Perf 
| where CounterName == "Free Megabytes"
| extend FreeGB = CounterValue / 1000
       , FreeKB = CounterValue * 1000


// Could use to repeat a column
Perf 
| where CounterName == "Free Megabytes"
| extend FreeGB = CounterValue / 1000
       , FreeMB = CounterValue 
       , FreeKB = CounterValue * 1000


// Can also use with strcat to create new string columns
Perf
| extend ObjectCounter = strcat(ObjectName, " - ", CounterName) 


//------------------------------------------------------------------------------
// Project
//------------------------------------------------------------------------------

// Project allows you to select a subset of columns
Perf
| project ObjectName
        , CounterName 
        , InstanceName 
        , CounterValue 
        , TimeGenerated 


// Combine Project with Extend
Perf
| where CounterName == "Free Megabytes"
| project ObjectName
        , CounterName 
        , InstanceName 
        , CounterValue 
        , TimeGenerated 
| extend FreeGB = CounterValue / 1000
       , FreeMB = CounterValue 
       , FreeKB = CounterValue * 1000


// You can use extend prior to project to calculate on a field
// you don't want in the final output (Counter Value omitted)
Perf
| where CounterName == "Free Megabytes"
| extend FreeGB = CounterValue / 1000
       , FreeMB = CounterValue 
       , FreeKB = CounterValue * 1000
| project ObjectName
        , CounterName 
        , InstanceName 
//------------------------------------------------------------------------------

// Returns a list of deduplicated values for columns from the input dataset
Perf 
| distinct ObjectName, CounterName


// Distinct can be used to limit a result set 
// Get a list of all sources that had an error event
Event
| where EventLevelName == "Error"
| distinct Source


//------------------------------------------------------------------------------
// Top
//------------------------------------------------------------------------------

// Top returns the first N rows of the dataset when the dataset is sorted by
// the "by" clause.
Perf
| top 20 by TimeGenerated desc


// When we combine what we've learned to produce something useful. 
// Here we're going to get a list of computers that are low on disk space
Perf
| where CounterName == "Free Megabytes"  // Get the Free Megabytes
    and TimeGenerated >= ago(1h)         // ...within the last hour
| project Computer                       // For each return the Computer Name
        , TimeGenerated                  // ...and when the counter was generated
        , CounterName                    // ...and the Counter Name
        , FreeMegabytes=CounterValue     // ...and rename the counter value
| distinct Computer                      // Now weed out duplicate rows as
         , TimeGenerated
         , CounterName                   // ...the perf table will have
         , FreeMegabytes                 // ...multiple entries during the day
| top 25 by FreeMegabytes asc            // Filter to most critical ones
