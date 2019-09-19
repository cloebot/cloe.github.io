---
layout: post
title: Append raw data to existing excel template
tags: [python, excel, pandas]
comments: true
---

When you report data to your team, you want to produce the report pretty enough to make your manager(s) happy. Or sometimes you need statistic calculation based on raw data you have.

In this posting, we will walk through how to convert the raw data (txt file or excel file) to an existing excel template without overwriting the existing format.

Before the start, my laptop has not installed any python library, I am going to walk through from installing library to the finish line.


### My laptop environment
- OS – ubuntu 18.04 (Window10 with WSL)
- Python3 (3.6.7)

### Dependencies:​ pandas, openpyxl

# Let’s start it!

I decided to use [Pandas](https://pandas.pydata.org/). This is an open source, data analysis tool for Python. They provide useful library working with Microsoft Excel. And using openpxyl, we are able to read/write Excel files. To begin with, we need to install Pandas and openpyxl for our dependencies.


## Install Pandas:
`sudo apt-get install python3-pandas`


## Install openpyxl
`sudo apt-get install python3-openpyxl`


Thanks for stack overflow user, I found [this helper function](https://stackoverflow.com/questions/20219254/how-to-write-to-an-existing-excel-file-without-overwriting-data-using-pandas) perfectly matches my requirement:

```python
def append_df_to_excel(filename, df, sheet_name='Sheet1', startrow=None,
                       truncate_sheet=False,
                       **to_excel_kwargs):
    """
    Append a DataFrame [df] to existing Excel file [filename]
    into [sheet_name] Sheet.
    If [filename] doesn't exist, then this function will create it.

    Parameters:
      filename : File path or existing ExcelWriter
                 (Example: '/path/to/file.xlsx')
      df : dataframe to save to workbook
      sheet_name : Name of sheet whi​ch will contain DataFrame.
                   (default: 'Sheet1')
      startrow : upper left cell row to dump data frame.
                 Per default (startrow=None) calculate the last row
                 in the existing DF and write to the next row...
      truncate_sheet : truncate (remove and recreate) [sheet_name]
                       before writing DataFrame to Excel file
      to_excel_kwargs : arguments which will be passed to `DataFrame.to_excel()`
                        [can be dictionary]

    Returns: None
    """
    from openpyxl import load_workbook

    import pandas as pd

    # ignore [engine] parameter if it was passed
    if 'engine' in to_excel_kwargs:
        to_excel_kwargs.pop('engine')

    writer = pd.ExcelWriter(filename, engine='openpyxl')

    # Python 2.x: define [FileNotFoundError] exception if it doesn't exist
    try:
        FileNotFoundError
    except NameError:
        FileNotFoundError = IOError


    try:
        # try to open an existing workbook
        writer.book = load_workbook(filename)

        # get the last row in the existing Excel sheet
        # if it was not specified explicitly
        if startrow is None and sheet_name in writer.book.sheetnames:
            startrow = writer.book[sheet_name].max_row

        # truncate sheet
        if truncate_sheet and sheet_name in writer.book.sheetnames:
            # index of [sheet_name] sheet
            idx = writer.book.sheetnames.index(sheet_name)
            # remove [sheet_name]
            writer.book.remove(writer.book.worksheets[idx])
            # create an empty sheet [sheet_name] using old index
            writer.book.create_sheet(sheet_name, idx)

        # copy existing sheets
        writer.sheets = {ws.title:ws for ws in writer.book.worksheets}
    except FileNotFoundError:
        # file does not exist yet, we will create it
        pass

    if startrow is None:
        startrow = 0

    # write out the new sheet
    df.to_excel(writer, sheet_name, startrow=startrow, **to_excel_kwargs)

    # save the workbook
    writer.save()
```

First step is to create [dataFrame](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.html)(Pandas data structure) out of my raw data. My raw data is seperated by ‘/t’ and line changes every ‘/n’. So in order to convert my txt file to excel, it should be:


```python
df = pd.read_csv("details.txt", sep="\t", header=None, skiprows=1)
```

I also skip the first row. This is header information but in my case, the header is already included in the template.


Next, change the output file permission to read/write for the owner, if the file already exists (I modified helper function little bit)

```python
append_df_to_excel(template_status, output_status, df, sheet_name, index=False, startrow=4)
```


Finally, change your output file back to read-only mode for your groups

```python
os.chmod(output_status, S_IREAD|S_IRGRP|S_IROTH)
```


You need to import os and stat to use above function

```python
from stat import S_IREAD, S_IRGRP, S_IROTH
import os
```

Now my result looks pretty neat, and all excel formulas (statistics and coloring) are perfectly along with my raw data!

![result-excel](/img/appendDataResult.png)
