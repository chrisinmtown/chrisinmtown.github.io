# Data Profiling in Python

### 12 Dec 2014

This blog entry offers some Python tools for profiling raw data.

At the time I was working with a customer to help them understand just
exactly what data they are receiving, what it means and how they might
best take advantage of it. Some is big data (hundreds of millions of
rows in daily deliveries of comma-separated value files), and some is
little data (thousands of rows in monthly deliveries of excel
workbooks). So I needed some basic tools to gain preliminary insights
into the files.  I had been learning python so I chose that
to build the scripts.

Data profiling?  Ok this is a four-dollar word that just means
figuring out the longest and shortest string values, the greatest and
least number values, the number of unique values and their
frequencies, and so on.  Getting this little collection of descriptive
statistics on a CSV or spreadsheet file can be remarkably insightful.
This is especially useful to do a quick check of fields (columns) that
are supposed to have a restricted set of values, or never be empty.

Of course this bit of data profiling could easily be done with a very
small handful of SQL queries.  But lately I've been hit repeatedly
with data long before it goes thru any extract-transform-load (ETL)
tools, so I get to analyze raw files.

See below for these scripts:

* profile_csv.py, a script that uses standard Python library
to iterate over a comma-separated value file.
* profile_excel.py, a script that uses the (immensely useful) xlrd
  library to iterate over an Excel workbook, either old or new
  format. You can get xlrd from https://pypi.python.org/pypi/xlrd
*  tablestat.py, a couple of Python classes that support gathering
  descriptive statistics from rows of tabular data.

I know I should be using Python 3, but my customer is still stuck on
version 2.  And admittedly these scripts are way, way wrong for data
that's hundreds of millions of rows, they're just way too slow.  Still
these have been a useful tool in my bag.  If these help you please
drop me a line.


## profile_csv.py

```
# Imports from the standard Python 2.7 library
from __future__ import print_function
import csv
import datetime
import getopt
import sys

# Local import - tablestat.py defines class TableStat etc.
import tablestat 

'''
Script to profile the values in a table read from a CSV file.
Keeps relatively little data so should be able to handle very large inputs.
'''
 
def profile_csv(header_skip, table, unique_max, file_obj):
    '''
    Reads a CSV file using csv.reader
    '''
    ts = None
    rownum = 0
    column_names = []
    reader = csv.reader(file_obj)
    for row in reader:
        # skip header rows as directed, possibly zero
        if rownum < header_skip:                 
            # First ensure list is the right length
            while len(column_names) < len(row):
                column_names.append("")
            # catenate values to form column names
            column_names = [ column_names[i] + str(row[i]) for i in xrange(len(row)) ]
            # detect the last header row           
            if rownum + 1 == header_skip:
                # instantiate the object
                ts = tablestat.TableStat(unique_max, column_names)
        else:
            # instantiate the stat collector bocs there were no header rows
            if ts is None and header_skip == 0:
                ts = tablestat.TableStat(unique_max, [])
            # analyze this data row (it's not a header)
            # SAMPLE 1%
            #if rownum % 100 == 0:
            ts.analyze_row(row)
        # finally increment the row number
        rownum += 1
        # be verbose during very large jobs
        if rownum % 1000000 == 0:
            print("%s row %d" % (datetime.datetime.now(), rownum), file=sys.stderr)
            sys.stderr.flush()
    # for all rows
    if rownum == 0:
        print("No rows read")
    elif table: 
        ts.print_report_thead("")
        ts.print_report_tbody("")
    else: 
        ts.print_report()
 
def usage():
    '''
    Prints a usage message and exits.
    '''
    print('profile_csv.py [options] [file.csv]')
    print('Reads from stdin if no file is given.')
    print('Options:')
    print('   -h header row skip count (default 1)')
    print('   -t tabular format report (default no)')
    print('   -u unique-limit (default 20)')
    sys.exit()

def main(args):
    '''
    Parses command-line arguments and profiles the named file.
    '''
    try:
        opts, args = getopt.getopt(args, "h:tu:")
    except getopt.GetoptError:
        usage()
    # default values
    hskip = 1
    table = False
    umax = 20
    for opt, optarg in opts:
        if opt in ("-h"):
            hskip = int(optarg)
        elif opt in ("-t"):
            table = True  
        elif opt in ("-u"):
            umax = int(optarg)
        else:
            usage()
    if len(args) == 1:
        print("Reading from file " + args[0],file=sys.stderr)
        with open(args[0]) as file_obj:
            profile_csv(hskip, table, umax, file_obj)
    else:
        print("Reading from stdin", file=sys.stderr)
        profile_csv(hskip, table, umax, sys.stdin)

# Pass all params after program name to our main
if __name__ == "__main__":
    main(sys.argv[1:])
```

## profile_excel.py

```
# Imports from the standard Python 2.7 library
import getopt
import sys

# xlrd, see http://www.python-excel.org and https://pypi.python.org/pypi/xlrd
# developed with version 0.9.3
from xlrd import open_workbook

# Local import - tablestat.py defines class TableStat etc.
import tablestat 

'''
Script to profile the values in sheets within an Excel file (xls or xlsx).
'''
 
def profile_excel(header_skip, table, unique_max, file_name, sheet_index):
    '''
    Reads a XLS file using xlrd
    Uses on-demand features to reduce memory requirements.
    TODO: Send date-time values as type datetime.datetime (not float)
    '''
    ts = None
    # detect failure to do anything
    found_sheet = False
    with open_workbook(file_name, on_demand=True) as wb_obj:
        sheet_names = [ n for n in wb_obj.sheet_names() ]
        # print("Sheet names: " + ",".join(sheet_names))
        for idx in xrange(len(sheet_names)):
            if sheet_index is not None and sheet_index != idx:
                continue
            s = wb_obj.sheet_by_name(sheet_names[idx])
            column_names = []
            for rownum in range(s.nrows):
                # get the row's values as a regular list
                row = [ s.cell(rownum,col).value for col in range(s.ncols) ]
                # skip header rows as directed, possibly zero
                if rownum < header_skip:
                    # gather header contents to use as cell names
                    # First ensure list is the right length
                    while len(column_names) < len(row):
                        column_names.append("")
                    column_names = [ column_names[i] + str(row[i]) for i in xrange(len(row)) ]
                    # detect the last header row           
                    if rownum + 1 == header_skip:
                        # instantiate the stat collector
                        ts = tablestat.TableStat(unique_max, column_names)
                else:
                    # special case for header-free inputs
                    if ts is None and header_skip == 0:
                        ts = tablestat.TableStat(unique_max, [])
                    # this is a data row (not a header), analyze it
                    ts.analyze_row(row)
            # for all rows
            # Free some memory
            s = None
            wb_obj.unload_sheet(sheet_names[idx])
            # print report for this sheet
            if table: 
                # emit header when the first sheet is found (a bit of a hack)
                if not found_sheet: ts.print_report_thead("Sheet name,Sheet index,")
                ts.print_report_tbody("%s,%d," % (sheet_names[idx], idx))
            else: 
                print ("---Begin sheet: '%s' (index %d)---" % (sheet_names[idx], idx))
                ts.print_report()
                print ("---End sheet: '%s' (index %d)---" % (sheet_names[idx], idx))
            # If we got here, we found a sheet.
            found_sheet = True
        # for all sheets
    # with
    # warn on bad arguments
    if not found_sheet:
        print("Failed to find sheet at index %d" % sheet_index)
        
def usage():
    '''
    Prints a usage message and exits.
    '''
    print('profile_excel.py [options] file.xls | file.xlsx')
    print('Options:')
    print('   -h header row skip count (default 1)')
    print('   -s sheet-index (default all)')
    print('   -t tabular format report (default no)')
    print('   -u unique-limit (default 20)')
    sys.exit()

def main(args):
    '''
    Parses command-line arguments and profiles the named file.
    '''
    try:
        opts, args = getopt.getopt(args, "h:s:tu:")
    except getopt.GetoptError:
        usage()
    # default values
    hskip = 1
    table = False
    sheetidx = None
    umax = 20
    for opt, optarg in opts:
        if opt in ("-h"):
            hskip = int(optarg)
        elif opt in ("-s"):
            sheetidx = int(optarg)
        elif opt in ("-t"):
            table = True  
        elif opt in ("-u"):
            umax = int(optarg)
        else:
            usage()
    if len(args) != 1:
        usage()
    profile_excel(hskip, table, umax, args[0], sheetidx)

# Pass all params after program name to our main
if __name__ == "__main__":
    main(sys.argv[1:])
```

## tablestat.py

```
'''
Created on Nov 24, 2014
'''

# future must be first
from __future__ import print_function
import datetime

# Constant values for speedy comparisons.
# A pseudo enum is clearer but runs lots of code.
datatype_unknown = -1
datatype_charstring = 0
datatype_digitstring = 1
datatype_number = 2
datatype_date = 3
datatype_mixed = 4

def get_datatype_name(d):
    '''
    Translates integers to strings for reporting.
    '''
    if d == datatype_charstring:     return "Charstring"
    elif d == datatype_digitstring:  return "Digitstring"
    elif d == datatype_number:       return "Number"
    elif d == datatype_date:         return "Date"
    elif d == datatype_mixed:        return "Mixed"
    else:                            return "Unknown"
    
# Inherits only from object
class TableStat(object):
    '''
    Provides methods to gather and report descriptive statistics on a 
    table of strings; for example, useful to profile a CSV file.
    Also handles integer, float and None values.
    
    Designed to be instantiated with a list of column names, but can
    also be created with an empty list; it will then self-assign names.
    
    Combines count of "None" values with count of empty string values.
    
    This code has been tuned for performance to handle large inputs.
    On a 3.1GHz win7 PC this requires about 1 min / 1 million rows,
    depending on the number of columns of course.

    Useful attributes: 
        unique_max (integer)
        row_count (integer)
        stats (list of ColumnStat objects)
        
    Profiled with "-m cProfile" arguments to python
    '''
    
    def __init__(self, unique_max_count, column_list):
        '''
        Constructor accepts an ORDERED list of column names.
        If the list is empty, assigns names as it does.
        '''
        # validate the input arguments
        if not isinstance(unique_max_count, int):
            raise Exception("Expected int but received %s" % type(unique_max_count))
        if not isinstance(column_list, list):
            raise Exception("Expected list but received %s" % type(column_list))
        # Keep the limit on unique values
        self.unique_max = unique_max_count
        # Number of rows seen
        self.row_count = 0
        # List of stat-collection objects, one per column
        self.stats = [ ColumnStat(i, column_list[i], self.unique_max) for i in xrange(len(column_list)) ]
            
    def analyze_row(self, data_list):
        '''
        Gathers statistics from the ORDERED list of data, which must match
        match the order and count of columns given to constructor.
        '''
        self.row_count += 1
        # compute length once, not repeatedly
        datalen = len(data_list)
        # Extend for wider-than-expected rows; nothing to do for narrow rows.
        if len(self.stats) < datalen:
            # Don't warn if column names began as an empty list
            if  len(self.stats) > 0:
                print("Warning: input row %d has %d columns but expected %d" % (self.row_count, datalen, len(self.stats)))
            while len(self.stats) < datalen:
                # Grow the list of column stat objects to allow
                # extra columns, or starting with no columns defined
                self.stats.append(ColumnStat(len(self.stats), None, self.unique_max))
        # Analyze each field in this row
        for i in xrange(datalen):
            self.stats[i].analyze_value(data_list[i])

    def print_report(self):
        '''
        Prints report on all columns to stdout, one result per line.
        This is convenient for humans.
        '''
        print("Row count = %d" % self.row_count)
        print("Note: unique value limit = %d" % self.unique_max)
        for i in xrange(len(self.stats)):
            self.stats[i].print_report()
    
    def print_report_thead(self, prefix):
        '''
        Prints header for column-oriented report.
        Prefix is used for additional column heads.
        '''
        self.stats[0].print_report_head(prefix)
        
    def print_report_tbody(self, prefix):
        '''
        Prints body of column-oriented report.
        Prefix is used for additional data columns.
        '''
        for i in xrange(len(self.stats)):
            self.stats[i].print_report_row(prefix)

# Inherits only from object
class ColumnStat(object):
    '''
    Gathers and reports descriptive statistics on a 
    collection of values, such as a single column in a table.
    Constructor takes index, name, unique_max limit.
    
    Useful attributes:
        name (string)
        datatype (inferred)
        empty (count of empty values)
        nonempty (count of non-empty values)
        values (set of unique values, up to limit)
		freqs (dict of unique values and their frequencies)
        minval, maxval (minimum and maximum numeric values)
        minlen, maxlen (minimum and maximum string lengths)
    '''

    def __init__(self, col_index, col_name, unique_max):
        # Keep the index & name
        self.index = col_index
        self.name = col_name
        # Limit set size to this maximum
        self.unique_max = unique_max
        # Datatype is inferred for strings
        self.datatype = datatype_unknown
        # Number of None, empty string or all-whitespace entries
        self.empty = 0 
        # Number of non-empty entries
        self.nonempty = 0
        # constants used as sentinels
        self.minsentinel = 999999999
        self.maxsentinel = -1
        # min and max lengths for strings
        self.minlen = self.minsentinel
        self.maxlen = self.maxsentinel
        # min and max values for numbers
        self.minval = self.minsentinel
        self.maxval = self.maxsentinel        
        # min and max values for dates
        self.mindate = None
        self.maxdate = None        
        # Unique value frequencies (size is limited)
        self.freqs = {}
        # set when freqs grows too long
        self.freqsfull = False
    
    def analyze_value(self, value):
        '''
        Analyzes a new value (i.e., new row) for the column.
        
        This is the critical method for performance tuning.
        '''
        # Track unique values/frequencies, limited by unique_max.
        if not self.freqsfull:
            if len(self.freqs) < self.unique_max: 
                # use get method's default value feature
                self.freqs[value] = self.freqs.get(value, 0) + 1
            else:
                # set flag in hope that boolean test is very fast
                self.freqsfull = True
            
        # Test for type
        if value is None:
            # This is not really expected, but don't blow up.
            self.empty += 1
        # this is a python 2.x solution; doesn't work in 3
        elif isinstance(value, basestring):
            # It has some kind of string.            
            if value == "" or value.isspace():
                self.empty += 1
                # Don't infer type based on a zero-length string.
                # TODO: possibly turn off digitstring on non-zero whitespace?
            else: 
                self.nonempty += 1
                # infer type of data within the string
                if value.isdigit():
                    # this value is only numbers
                    if self.datatype == datatype_digitstring:
                        # this is the most common case, no need to look further
                        pass
                    elif self.datatype == datatype_unknown:
                        # first sighting
                        self.datatype = datatype_digitstring
                    elif self.datatype != datatype_charstring and self.datatype != datatype_digitstring:
                        # previously had non-string value, so mark as mixed
                        self.datatype = datatype_mixed
                else:
                    # not digits
                    if self.datatype == datatype_charstring:
                        # this is the most common case, no need to look further
                        pass
                    elif self.datatype == datatype_unknown or self.datatype == datatype_digitstring:
                        # first sighting, or first presence of chars
                        self.datatype = datatype_charstring
                    elif self.datatype != datatype_charstring:
                        # previously had non-string type
                        self.datatype = datatype_mixed
                # done with type
            # track min/max length of the string
            # calculate length once, not 2+ times
            strlen = len(value)
            if strlen < self.minlen: self.minlen = strlen
            if strlen > self.maxlen: self.maxlen = strlen
        
        elif isinstance(value, int) or isinstance(value, long) or isinstance(value, float):
            # It's a proper number.
            self.nonempty += 1
            # Note type
            if self.datatype == datatype_unknown: self.datatype = datatype_number
            elif self.datatype != datatype_number: self.datatype = datatype_mixed
            # Store min/max numeric values
            if value < self.minval: self.minval = value
            if value > self.maxval: self.maxval = value
            
        elif isinstance(value, datetime.datetime):
            # It's a date-time value; first seen from XLSX via openpyxl
            self.nonempty += 1
            # Note type
            if self.datatype == datatype_unknown: self.datatype = datatype_date
            elif self.datatype != datatype_date: self.datatype = datatype_mixed
            # Store min/max date values
            if self.mindate is None or value < self.mindate: self.mindate = value
            if self.maxdate is None or value > self.maxdate: self.maxdate = value
            
        else:
            # Tabular data should not have non-scalar values like list, etc.
            raise Exception("Cannot profile type " + str(type(value)))
            
            # not a string
        
    def print_report(self):
        '''
        Prints field report to stdout, one result per line.
        '''
        print("Column '%s' (index %d)" % (self.name, self.index))
        print("\tData type      = %s" % get_datatype_name(self.datatype))
        print("\tEmpty count    = %d" % self.empty)
        print("\tNonempty count = %d" % self.nonempty)
        print("\tDensity        = %f" % self.get_density())
        if self.datatype == datatype_charstring or self.datatype == datatype_digitstring:
            print("\tMax length str = %s" % self.maxlen)
            print("\tMin length str = %s" % self.minlen)
        if self.datatype == datatype_number:
            print("\tMax number     = %s" % self.maxval)
            print("\tMin number     = %s" % self.minval)
        if self.datatype == datatype_date:
            print("\tMax date       = %s" % self.maxdate)
            print("\tMin date       = %s" % self.mindate)
        # Don't just echo the max value count when it's exceeded
        if len(self.freqs) < self.unique_max:
            print("\tUnique count   = %d" % len(self.freqs))
            # emit dictionary contents sorted by key
            print("\tUnique values  = %s" % "{" + ", ".join("%r: %r" % (key, self.freqs[key]) for key in sorted(self.freqs)) + "}")
        else:
            print("\tUnique count   > %d" % self.unique_max);
    
    def get_density(self):
        return (self.nonempty / float(self.empty + self.nonempty))
    
    def print_report_head(self, prefix):
        '''
        Prints the column heads for a tabular report to stdout.
        Use in conjunction with print_report_row.
        Optionally adds a prefix, which supports sheet name and index.
        '''            
        print(prefix + "Column name,Column index,Data type,Empty count,Nonempty count,Density,"
              + "Max length str,Min length str,Max number,Min number,Max date,Min date,"
              + "Unique count,Unique values")  
        
    def print_report_row(self, prefix):
        '''
        Prints field report to stdout, one result per column.
        Optionally adds a prefix, which supports sheet name and index.
        '''
        # Avoid a hopelessly wide line.
        if len(self.freqs) < self.unique_max:
            myuniques = str(len(self.freqs))
            # emit dictionary contents sorted by key
            myfreqs = "{" + ", ".join("%r: %r" % (key, self.freqs[key]) for key in sorted(self.freqs)) + "}"
        else:
            myuniques = "> " + str(self.unique_max)
            myfreqs = "Unknown"
        print(prefix
              + "%s," % self.name
              + "%d," % self.index
              + "%s," % get_datatype_name(self.datatype) 
              + "%d," % self.empty
              + "%d," % self.nonempty
              + "%f," % self.get_density()
              + "%s," % (self.maxlen if self.maxlen != self.maxsentinel else None) # emit as string to allow None
              + "%s," % (self.minlen if self.minlen != self.minsentinel else None)
              + "%s," % (self.maxval if self.maxval != self.maxsentinel else None)
              + "%s," % (self.minval if self.minval != self.minsentinel else None)
              + "%s," % self.maxdate
              + "%s," % self.mindate
              + "%s," % myuniques
              + '"%s"' % myfreqs     # surround with quotes
            )

# Basic tests
# TODO: How to generate a date-time value?
if __name__ == "__main__":
    cols = [ "Empties", "Numbers", "Strings", "Digitstring" ]
    ts = TableStat(unique_max_count = 5, column_list = cols)
    ts.analyze_row([None , None, None   , None  ])
    ts.analyze_row(["   ", 1   , "hi"           ])
    ts.analyze_row(["\t" , 2   , "world", "456", "bonus" ])
    ts.analyze_row([None , 3.0 , "bar"  , "789" ])
    ts.analyze_row([None , 4.0 , "bar"  , "012" ])
    print("Generating tall report:")
    ts.print_report()
    print("Generating wide report:")
    ts.print_report_thead("")
    ts.print_report_tbody("")
    # This tests constructor input validation
    # bogus = TableStat("hi")
```

Please leave comments [at the github repo](https://github.com/chrisinmtown/chrisinmtown.github.io)
