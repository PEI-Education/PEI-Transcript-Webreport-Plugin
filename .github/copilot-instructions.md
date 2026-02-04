# PEI Transcript PowerSchool Plugin - AI Agent Guide

## Project Overview

This is a PowerSchool SIS plugin that generates official Prince Edward Island student transcripts. The plugin provides a web-based report accessible through PowerSchool's admin interface, allowing users to print transcripts for selected students.

### Key Technologies
- **PowerSchool SIS Plugin Architecture** (ZIP-based deployment)
- **PowerQueries** - Oracle SQL-based data queries for the PowerSchool API
- **PowerSchool HTML (PS-HTML)** - Server-side templating language with tags like `~[text:...]` and `~(variable)`
- **Handlebars.js** - Client-side templating for transcript generation
- **Vanilla JavaScript** with jQuery (`$j`) - Frontend interactivity
- **Oracle SQL** - Database queries against PowerSchool schema

### Plugin Structure
```
plugin.xml                          # Plugin manifest (name, version, publisher)
queries_root/                       # PowerQuery definitions (API data sources)
  └── pei.education.transcript.named_queries.xml
WEB_ROOT/                          # Web-accessible files
  └── admin/
      └── pei_transcript/
          ├── pei_transcript_prefs.html  # Preferences/configuration page
          ├── pei_transcript.html        # Main transcript generation page
          ├── css/                       # Styling
          └── image/                     # Assets (logos)
permissions_root/                   # API endpoint permissions
  └── pei.education.permission_mappings.xml
MessageKeys/                        # Internationalization (English/French)
  ├── pei_transcript.US_en.properties
  └── pei_transcript.CA_fr.properties
pagecataloging/                     # Navigation menu definitions
  └── pei_transcript.json
```

## Architecture & Data Flow

### High-Level Flow
1. **User Selection**: Admin user selects students in PowerSchool and navigates to "PEI Student Transcript"
2. **Preferences Page** (`pei_transcript_prefs.html`): User configures report options (sort order, signature, current courses, etc.)
3. **Transcript Generation** (`pei_transcript.html`):
   - Three PowerQuery API calls fetch data in parallel:
     - `demographic_data`: Student demographics and graduation info
     - `historical_grades`: Completed course history  
     - `current_courses`: In-progress and upcoming courses
   - JavaScript processes and combines the data
   - Handlebars template renders 2-page transcripts (front + reverse with definitions)
   - Browser displays printable/PDF-ready output

### PowerQuery Architecture (Critical)
PowerQueries are the **data source layer** between the plugin and PowerSchool's database. They:
- Define SQL queries with parameters
- Enforce Data Restriction Framework (DRF) security
- Support both internal API use (web pages) AND Data Export Manager (DEM)
- Use `<@restricted_table>` tags to respect user selection scope

**Three PowerQueries in this plugin:**
1. `pei.education.reporting.transcript.demographic_data` - Student details, school info, grad status, ESAP enrollment
2. `pei.education.reporting.transcript.historical_grades` - StoredGrades with credit types
3. `pei.education.reporting.transcript.current_courses` - Active CC enrollments with optional interim marks

### Security & Permissions
- **Permission Mapping** (`permissions_root/pei.education.permission_mappings.xml`): Links the prefs page to PowerQuery POST endpoints
- **DRF (Data Restriction Framework)**: PowerQueries use `<@restricted_table>` to limit results based on user role (admin/teacher/parent/student)
- **Field Level Security**: Users must have access to columns referenced in `<column column="...">` mappings

## Critical PowerQuery Requirements (Common Pitfalls)

### ⚠️ Requirements for Data Export Manager (DEM) Visibility

PowerQueries will NOT appear in DEM if they violate these rules:

1. **NO PS-HTML tags in SQL** - Tags like `~[text:...]` or `~(variable)` work in web pages but **break PowerQueries for DEM**
   - ❌ BAD: `when 'EN' then '~[text:psx.htmlc.admin.type_EN]'`
   - ✅ GOOD: `when 'EN' then 'English'`

2. **NO PS-HTML tags in `default` arg values** - Using `default="~(curschoolid)"` prevents DEM visibility
   - ❌ BAD: `<arg name="curyearid" default="~(curyearid)"/>`
   - ✅ GOOD: `<arg name="curyearid" default="33"/>` or omit `default`

3. **Column count MUST match SQL output** - Number of `<column>` elements must exactly equal SELECT columns
   - Count your SQL SELECT columns carefully!

4. **Column mappings** - The `column="table.field"` attribute MUST reference real PowerSchool core tables/fields
   - The text between `<column>` tags is arbitrary (just display names)
   - Column field types should match the data type being returned (e.g., `to_char(dob)` returns string, map to VARCHAR field)
   - When in doubt: use `column="students.id"` for everything

5. **Unique column aliases** - Every SELECT column needs a unique `AS alias` name

6. **ORDER BY with table aliases** - Always qualify: `ORDER BY students.lastfirst` not just `lastfirst`

7. **CDATA blocks** - Always wrap SQL in `<![CDATA[...]]>`

### PowerQuery XML Example
```xml
<query name="pei.education.reporting.transcript.demographic_data" 
       coreTable="students" flattened="true" tags="transcript">
  <summary>PEI Transcript: Student Details</summary>
  <description>Retrieves student transcript data</description>
  
  <args>
    <!-- NO default="~(curschoolid)" for DEM compatibility -->
    <arg name="curschoolid" column="students.schoolid" required="false"/>
  </args>
  
  <columns>
    <!-- Column count MUST match SQL SELECT count -->
    <column column="students.last_name">Demographics.Legal_Last_Name</column>
    <column column="students.first_name">Demographics.First_Name</column>
    <!-- ... 27 total columns to match 27 SELECT columns -->
  </columns>
  
  <sql><![CDATA[
    select
      scf.pscore_legal_last_name legal_last_name,  -- Must have unique alias
      scf.pscore_legal_first_name legal_first_name,
      -- NO ~[text:...] tags in SQL!
      case when s.gradstatus = 'Y' then 1 else null end as graduated
    from (select dcid from students 
          where <@restricted_table table="students" selector="selectedstudents"/>) sel
    join students s on s.dcid = sel.dcid
    order by s.lastfirst  -- Always use table alias
  ]]></sql>
</query>
```

## Developer Workflows

### Plugin Installation/Testing
1. **Package Plugin**: Zip the root folder contents (not the folder itself)
2. **Install**: System > System Settings > Plugin Management Dashboard > Install
3. **Enable**: Check the plugin checkbox and save
4. **Test**: Make student selection, navigate to "PEI Student Transcript"

### Debugging PowerQueries
- **Test SQL in SQL Developer**: Use `:argname` for parameters
- **Check PowerQuery availability**: Data Export Manager should list queries
- **Console Errors**: Browser DevTools Network tab shows 400/500 errors with details
- **Column count mismatch**: Most common cause of 400 errors

### Testing Workflow
1. In PowerSchool test environment, select 1-5 students
2. Navigate: Special Functions > Group Functions > PEI Student Transcript
3. Configure options and generate
4. Verify:
   - Data loads without errors
   - Transcripts render correctly
   - Current courses appear if selected
   - French translations work (change browser locale)

### Making Changes
1. **PowerQuery changes**: Edit `queries_root/*.xml`, re-zip, reinstall plugin
2. **HTML/JS changes**: Edit in `WEB_ROOT/`, re-zip, reinstall
3. **Always increment version** in `plugin.xml` when deploying

## Project-Specific Conventions

### Naming Conventions
- PowerQuery names: `pei.education.reporting.transcript.<queryname>` (5-part reversed domain)
- Column aliases: `Demographics.Field_Name`, `Credits.Field_Name`
- File naming: Descriptive with underscores (`pei_transcript_prefs.html`)

### PowerSchool-Specific Patterns
- **jQuery alias**: PowerSchool uses `$j` not `$` to avoid conflicts
- **PS-HTML tags**: `~[text:key]` for i18n, `~(variable)` for server vars
- **Student selection**: Use `dofor=selection:selectedstudents` in API calls
- **Pagination**: PowerQueries support `pagesize=1500` parameter
- **Table extensions**: Use `u_def_ext_*` for custom fields

### Data Processing Pattern (JavaScript)
```javascript
// 1. Fetch data from 3 PowerQueries in parallel
const [students, credits, currentCourses] = await Promise.all([...fetches])

// 2. Process: filter/group courses, calculate totals
students.forEach(student => {
  student.courses = { completed: [], attempted: [], opc: [], esap: [] };
  student.rawCourses.forEach(course => {
    // Group by credit type, earned vs attempted
  });
});

// 3. Render with Handlebars in chunks (25 students at a time)
renderChunks(students, 25);
```

### Internationalization
- All user-facing text uses `~[text:key]` references
- Keys defined in `MessageKeys/*.properties` files
- English (US_en) and French (CA_fr) supported

## Important Documentation Files

- **`/Context/PowerQuery_docs_clean.md`**: Official PowerQuery syntax, features, version matrix
- **`/Context/PowerQuery_Common_Pitfalls.md`**: Undocumented issues, DEM errors, SQL gotchas ⚠️ READ THIS FIRST
- **`/Context/PowerQuery_DAT.md`**: Data Access Tags (for embedding queries in PS-HTML pages)
- **`/Context/Data_Restriction_Framework.md`**: User role-based data filtering
- **`/Context/pei.dell.transcript_data.named_queries.xml`**: Working example queries for reference

## Common Tasks & Solutions

### Adding a new data field to transcripts
1. Add column to PowerQuery SQL SELECT
2. Add matching `<column>` element (must match count!)
3. Update Handlebars template to display `{{ field_name }}`
4. Test: Check browser console for 400 errors

### Troubleshooting 400 errors on PowerQuery fetch
- **Column count mismatch**: Count SQL columns vs `<column>` elements
- **Missing required arg**: Check if query has `required="true"` args
- **Data type mismatch**: Ensure arg values match column data types
- **Browser Console**: Check request/response payloads in Network tab

### Making query work in Data Export Manager
1. Remove ALL `~[text:...]` tags from SQL (use literal strings)
2. Remove PS-HTML from arg `default` values
3. Verify column count matches SQL output
4. Test: Check if query appears in DEM after re-installing plugin

### Performance optimization
- **Batch rendering**: Renders 25 students at a time to avoid browser hang
- **Pagination**: Use `pagesize` parameter appropriately (1500 students max recommended)
- **Parallel fetches**: All 3 PowerQueries called simultaneously with `Promise.all()`

## Critical Gotchas

1. **Oracle `substr` position**: Position 0 is treated as 1, but inconsistent - use 1-based indexing
2. **Date fields as strings**: `to_char(dob, 'MM/DD/YYYY')` returns VARCHAR, not DATE
3. **PS-HTML evaluation order**: Server evaluates `~()` tags before browser sees page
4. **Plugin caching**: Sometimes need to disable/re-enable plugin, not just reinstall
5. **Selection scope**: `<@restricted_table>` respects user's student selection AND role permissions

## Version Control & Deployment

Plugin Versioning:
- **Current version**: 2026.1.4.5 (see `plugin.xml`)
- **Production version**: 2026.1.5

PowerSchool Versions:
- **Production**: PowerSchool SIS 25.9.0
- **Testing**: PowerSchool SIS 25.12.0

Deployment Steps:
- **Deployment**: Zip entire plugin directory, install via Plugin Management Dashboard
- **Always increment version number** in `plugin.xml` before deploying

## Useful Patterns

### Fetching PowerQuery with Arguments
```javascript
fetch(`/ws/schema/query/pei.education.reporting.transcript.current_courses?dofor=selection:selectedstudents&pagesize=12000`, {
  method: 'POST',
  credentials: 'same-origin',
  headers: {
    'Accept': 'application/json',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    "curyearid": ~(curyearid),  // Server evaluates to actual year
    "storecode": "Q1"
  })
});
```

### LEFT JOIN with Row Limiting (Common Pattern)
```sql
left join (
  select studentid, schoolid, 
         row_number() over (partition by studentid order by yearid desc) as rn 
  from ps_schoolenrollment 
  where schoolid < 700
) sed 
  on sed.studentid = students.id
  and sed.rn = 1
```

This guide should enable any AI agent to quickly understand the codebase architecture, avoid common PowerQuery pitfalls, and make productive changes to the transcript plugin.