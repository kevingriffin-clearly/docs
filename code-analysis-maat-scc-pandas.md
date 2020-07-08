# Code analysis with code-maat, scc, and pandas
### Kevin Griffin <kevin.griffin@clearly.ca>
### Site reliability
Written in  honor of [Your Code as a Crime Scene](https://pragprog.com/titles/atcrime/)

Scenario: You have taken over an application and are trying to figure out what areas of the code base might give you the most problems.

We'll be analyzing  the git log to see how often a peice of code gets committed to, and look at the relative sizes. 
We're using a very small sample tomcat application to analyze,

say : https://github.com/heroku/devcenter-embedded-tomcat


check out the code base:
```
git clone https://github.com/heroku/devcenter-embedded-tomcat
```
cd into the new directory and make a gitlog. I build the gitlog with maat in the name - because we will be using this with a tool called [code-maat](https://github.com/adamtornhill/code-maat)

```
devcenter-embedded-tomcat$ git log --pretty=format:'[%h] %aN %ad %s' --date=short --numstat --after=2010-01-01 >maat-git1.log
```
use code maat to pick out how often a file is revised, and save that as a csv:
(your maat location will probably be different from mine.)
```
java -jar ~/bin/code-maat-0.8.5-standalone.jar -c git -l maat-git1.log -a revisions > ../revisions.csv
```
So that gives a list of files that have experienced a lot of churn.

Next we'll need [scc](https://github.com/boyter/scc) - this is a tool that can parse out lines of code, do complexity analysis, all sorts of fun stuff.

Let's look at what files are large- we'll use that as  a proxy for complexity.
```
scc --by-file -c --no-cocomo  -f csv -o ../scc-tomcat-lines.csv
```
Now we can correleate these two files - code maat calls the filename "entity" and scc calls it "location". We can use that to combine the results.

Install pandas, ( pip3 install pandas ) if you don't already have it.
Fire up python:

```
python3
Python 3.7.4 (v3.7.4:e09359112e, Jul  8 2019, 14:54:52)
[Clang 6.0 (clang-600.0.57)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>>import pandas as pd
```

Now let's import those csvs as "dataframes" 
- think of them like spreadsheets - labeled columns and rows. - For more see [pandas-tutorial](https://www.datacamp.com/community/tutorials/pandas-tutorial-dataframe-python)
```
tomcat_freq = pd.read_csv("../revisions.csv")
tomcat_sizes = pd.read_csv("../scc-tomcat-lines.csv")
```
Let's confirm those look the way we think they should:
```
tomcat_freq.head()
tomcat_sizes.head()
```
Okay so let's rename the 'entity' column in the frequency dataframe to 'filename'. We're trying to creating a column we can index both tables against.
```
df = tomcat_freq.rename(columns={'entity': 'filename'})
list(df.columns)
```
And for the sizes data frame - the closest we have to filename is the "Location" column, so let's rename that.
```
df2 = tomcat_sizes.rename(columns={'Location': 'filename'})
```

Okay, let's merge the two:
```
right_merge = pd.merge(df, df2, on='filename', how='right')
```
And sort them first by n-revs (number of revisions) and then by lines of code. And We want the biggest first:
```
sorted_right_merge = right_nmerge.sort_values(by=['n-revs','Code'],ascending=False)
```
Checking the head of the dataframe to see if it looks right:
```
>>> sorted_right_merge.head()
                                  filename  n-revs   Language           Filename  Lines  Code  Comments  Blanks  Complexity
0                                  pom.xml      13        XML            pom.xml     67    67         0       0           0
1                                README.md      11   Markdown          README.md     24    14         0      10           0
2           src/main/java/launch/Main.java       7       Java          Main.java     81    77         0       4           0
3                               .gitignore       4  gitignore         .gitignore      7     5         1       1           0
4  src/main/java/servlet/HelloServlet.java       2       Java  HelloServlet.java     27    22         0       5           0
```
looking good. Save to csv.
```
sorted_right_merge.to_csv('../sorted_right_merge.csv')
```

For fun or future reference you can show your python session history with:
```
import readline; print('\n'.join([str(readline.get_history_item(i + 1)) for i in range(readline.get_current_history_length())])
```
But truly, this toolchain truly shines when used to look at large, older code bases. The classic example is look at the Hibernate project.
