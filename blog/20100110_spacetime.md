# Space verus Time in Java's ObjectOutputStream

### 10 Jan 2010

This past week I helped my colleague D. B. investigate a memory-use
issue in a component he wrote.  The component reads and sorts records
(basically database rows). Because the number of rows can be large, it
uses a disk-based mergesort once the count exceeds a user-defined
threshold in order to keep memory use in check.  Under the covers it
uses Java's ObjectOutputStream to write each chunk of sorted input and
ObjectInputStream to read those chunks while merging.  Well,
despite our best intention and design, the sorter component was using
far more memory than we expected, resulting in the dreaded
`java.lang.OutOfMemoryError: Java heap space` exception.  As a first
step I installed TPTP trying to localize the problem.  It showed me
tens of thousands of live char[] instances in use but refused to
report for me the origin of those instances, so that tool wasn't much
help.  Finally he hit upon the answer, namely that ObjectOutputStream
was not releasing references.  We thought that using writeUnshared()
would be enough but it was not.  The javadoc hints at this behavior,
but I guess does not state it quite as clearly as we would like.

The solution is to call the class's reset() method at suitable
intervals.  We experimented and found some significant performance
differences based on how often we call that method.  So I wrote a
little code (see below) and gathered some numbers.  The tables below
show results from processing an input CSV file of 544,222 lines and
size 86Mb, on a 2Ghz dual-core laptop running WinXP on an ordinary
disk drive.  All tests launched from Eclipse in debug mode with a Java
breakpoint set on the last executable line.  The interval column shows
how many records were written between each call to the object output
stream's reset() method.  The program reports its time in
milliseconds.  Memory numbers were gathered from the windoze taskmgr
"Mem Usage" column when the breakpoint was reached.

<table cellspacing="0" cellpadding="2" border="1">
<tr>
<th colspan=3">Using writeObject()</th>
<th colspan=3">Using writeUnshared()</th></tr>
</tr>
<tr>
<th>Interval</th><th>Time/MS</th><th>Memory/KB</th>
<th>Interval</th><th>Time/MS</th><th>Memory/KB</th>
</tr>

<tr>
<td align="right">1</td><td>    22,781</td><td>    10,460</td>
<td align="right">1</td><td>    23,093</td><td>    10,464</td>
</tr>

<tr>
<td align="right">5</td><td>    18,579</td><td> 10,468</td>
<td align="right">5</td><td>    18,531</td><td> 10,480</td>
</tr>

<tr>
<td align="right">10</td><td>    17,875</td><td>    10,496</td>
<td align="right">10</td><td>    17,907</td><td>    10,540</td>
</tr>

<tr>
<td align="right">50</td><td>    17,484</td><td>     10,524</td>
<td align="right">50</td><td>   17,375</td><td>  10,516</td>
</tr>

<tr>
<td align="right">100</td><td>    17,453</td><td> 11,128</td>
<td align="right">100</td><td>    17,500</td><td>    10,712</td>
</tr>

<tr>
<td align="right">1,000</td><td>19,781</td><td>    14,692</td>
<td align="right">1,000</td><td>18,782</td><td> 14,540</td>
</tr>

<tr>
<td align="right">10,000</td><td>20,329</td><td>21,368</td>
<td align="right">10,000</td><td>19,844</td><td>21,812</td>
</tr>
</table>


Calling reset at every write yields minimal memory use but it's huge
overkill that significantly increases the run time.  On the other end,
wait too long to call reset and the memory use blows up the JVM, and
the run time starts increases also (which I really didn't expect).

A person on the Sun forum (see the link in the code) pointed that this
undesired behavior pops up out when the object written out has
references within it.  In my first attempt at this assessment I simply
wrote out String objects read from the CSV file.  But in that
scenario, writeUnshared works perfectly, does what I expect, and the
program uses very little memory.  The way to reproduce the excessive
memory use problem in this little example is to split the lines and
write out the resulting array of string.  Then the numbers showed no
difference between the writeObject() and writeUnshared() methods.

The conclusion: calling reset at some interval is essential, and
there's a sweet spot at about every 100th write where the space/time
tradeoff is reasonable.  I didn't investigate whether the sweet spot
is really at 50 or 150; here's the code if you want to go after that
one. 

```
import java.io.BufferedReader;
import java.io.File;
import java.io.FileOutputStream;
import java.io.FileReader;
import java.io.ObjectOutputStream;
import java.util.Date;

/**
 * Program to measure the space/time tradeoff in performance
 * when using {@link ObjectOutputStream#writeObject(Object)}
 * and {@link ObjectOutputStream#writeUnshared(Object)}.
 * 
 * Also see:
 * http://bugs.sun.com/bugdatabase/view_bug.do?bug_id=4332184
 * http://bugs.sun.com/bugdatabase/view_bug.do?bug_id=4363937
 * http://forums.sun.com/thread.jspa?threadID=5419725
 * 
 */
public class SpaceVsTime {

    public static void main(String[] args) {
        
        if (args.length != 3) {
            System.err.println("Usage: SpaceVsTime csv-file object-file reset-interval");
            return;
        }

        File csvFile = new File(args[0]);
        File objFile = new File(args[1]);
        int recInterval = Integer.parseInt(args[2]);
        
        if (! csvFile.exists()) { 
            System.err.println("Not found or not a file: " + csvFile.getPath());
            return;
        }
        
        try {
            Date start = new Date();
            System.out.println(start + ": started with interval " + recInterval);

            int count = 0;
            BufferedReader bufRdr = new BufferedReader(
                    new FileReader(csvFile));
            ObjectOutputStream oos = new ObjectOutputStream(
                    new FileOutputStream(objFile));
            String line;
            while( (line = bufRdr.readLine()) != null) {
                String [] cols = line.split(",");
                oos.writeUnshared(cols);
                // oos.writeObject(cols);
                ++count;
                if (count % recInterval == 0)
                    oos.reset();
            }
            bufRdr.close();
            oos.close();            
            
            Date finish = new Date();
            System.out.println(finish + ": finished for " + count + " records");
             System.out.println("Execution time (ms): " + (finish.getTime() - start.getTime()));
             System.out.println();
        }
        catch (Exception ex) {
            ex.printStackTrace();
        }
    }
}
```
