A General Timeseries Class for Astropy
-------------------------------------

author: Stuart Mumford

date-created: 2016 March 24

date-last-revised: 2016 March 24

type: Standard Track

status: Discussion


Abstract
--------

The goal of a timeseries object in astropy is to provide a core set of
functionality that can be shared between astropy, affiliated packages and the
wider community.

The scope of this timeseries object is designed to remain focused on the core
functionality needed, and not to try and replace or re-implement the
functionality in the pandas library. The main motivation for this class over
using pandas is primarily to support the functionality of other parts of the
astropy package such as Time, Units and Coordinates packages.

This document proposes an implementation based on Table, with extra constraints
and specific functionality dedicated to timeseries type data.

Detailed description
--------------------

Many different areas of astrophysics have to deal with 1D timeseries data, i.e.
either sampling a continuous variable at fixed times or counting some events
binned into time windows. These types of applications require some basic
functionality:

#. Extending timeseries with extra rows
#. Concatenating multiple timeseries objects
#. Sorting
#. Slicing / selecting time ranges, i-index and column (indexing)
#. Re-binning and re-sampling timeseries
#. Interpolating to different time stamps.
#. Masking
#. Support for subtraction and addition.
#. Plotting and visualisation.

While this functionality is found in non-domain specific packages such as
pandas, a timeseries object in Astropy would also provide some functionality
which is more related to Astronomy:

#. Converting between time systems
#. Astropy unit support
#. Support variable width time bins.

It is proposed that this functionality be implemented via a subclass of the
``QTable`` class, with a few requirements specific to the timeseries class:

#. A 'Time' index column exists which enforces unique indexes.
#. The table is always sorted in terms of increasing time.


Binned Data vs Sampled Data
###########################

There are two different types of data that a timeseries class or classes would
have to support, sampled data and binned data. Sampled data is taken at a
precise time stamp whereas binned data is a number of counts contained within a
some time window. These two different types of timeseries data require different
handling of methods such as re-binning or re-sampling, as well as for sampled
data the width of the bins must be stored.

This APE proposes that we place the following restrictions on binned data:

#. Contiguious bins, i.e. the start of the i+1th bin is the end of the ith bin.
#. The width of the bins is stored in a second column containing ``TimeDelta`` objects.


Indexing
--------

Indexing by Time-Like Objects
#############################

When indexing using time-like values you get a TimeSeries returned.
A single index value can be used:

::
	single_row_timeseries = timeseries[datetime]


Which returns a Timeseries with a single row and the same columns as the original.

A range of datetimes:

::
	multi_row_timeseries = timeseries[datetime_start: datetime_end]


Which returns a TimeSeries truncated between the datetime_start (inclusive) and datetime_end (exclusive?).

*Note: neither datetime_start nore datetime_end need to be exact index value in the original Timeseries.*

If you input a time-like value not in the index then raise an IndexError.



	
Indexing by Integer (i-index)
#############################

Similar to time-like indexing either implicitly or using the .iloc attribute: ::
	single_row_timeseries = timeseries[i_integer] = timeseries.iloc[i_integer]

::
	multi_row_timeseries = timeseries[i_integer_start: i_integer_end] = timeseries.iloc[i_integer_start: i_integer_end]
  
With support of negative index values:

::
	single_row_timeseries = timeseries[-i_integer] = timeseries.iloc[-i_integer] = timeseries.iloc[-i_integer] = timeseries.iloc[-i_integer]
 
::
	multi_row_timeseries = timeseries[i_integer_start: -i_integer_end] = timeseries.iloc[i_integer_start: -i_integer_end]
	


Indexing by Column Name (String)
################################

When indexing using a String for the column name you get a Timeseries with the given column/s.

You can use a single column name:

::
	single_column_timeseries = timeseries[colname_string]
  
Which returns a Timeseries with a single column and the same rows/indices as the original.

You can use a multiple column:

::
	multi_column_timeseries = timeseries[colname_string1, colname_string2, colname_string3]
  
Which returns a Timeseries with a multiple columns (in the given order) and the same rows/indices as the original.

The names for the columns should be accessable, for example using the Table-like attribute colnames:

::
	timeseries.colnames

*Note: Pandas DataFrames have a similar columns attribute, but AstroPy Table have an attribute with this name that returns a TableColumns object.*
	

Indexing by Bolean Array
########################

You can select multiple disjoint rows using a 1D array of Boolean values (of same length as the number of TS rows), a Timeseries is returned with all the rows corresponding to True, as with Numpy Array, Pandas DataFrame and AstroPy Table:

::
	multi_row_timeseries = timeseries[boolean_array]

This is often used when filtering, for example based on value:

::
	multi_row_timeseries = timeseries[timeseries['col_b'] > 5.2]


Indexing Defaults
#################

It could be implmented that the choice of loc, iloc or column could be made by input variables, assuming the Timeseries index can only be time-like (not integer or float) and columns names are only allowed as Strings.

For strings we use the .column slicing function.

For time-like inputs we would choose to default to .loc (if we add time-like strings this could impinge of column name detection).

For integer inputs we use the .iloc slicing function.



Indexing Comparison with Pandas DataFrame
#########################################

The following are examples of similar functionality using the Pandas DataFrame:
::
  import pandas as pd
  import numpy as np
  a = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
  b = [10, 11, 12, 13, 14, 15, 16, 17, 18, 19]
  c = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j']
  dic_data = {'col_a': a, 'col_b': b, 'col_c': c}
  arr_index = pd.date_range('1/1/2011', periods=10, freq='2H')
  df_ts = pd.DataFrame(data=dic_data, index=arr_index)

You can slice using:
::
  pandas_series_row = df_ts.loc[df_ts.index[5]]  # where df_ts[df_ts.index[5]] fails

::
  pandas_series_row = df_ts.iloc[5] = df_ts[5]

::
	truncated_dataframe = df_ts.loc[df_ts.index[2]: df_ts.index[5]]

::
	truncated_dataframe = df_ts.loc[pd.to_datetime('2011-01-01 05:00:00'):pd.to_datetime('2011-01-01 08:00:00')] # values not in index.

::
	truncated_dataframe = df_ts.iloc[2:5] = df_ts[2:5]

::
	truncated_dataframe = df_ts.iloc[2:5:2] = df_ts[2:5:2]

Note: DF[start:end] = DF.iloc[start:end] is exclusive for end while DF.loc[start:end] is inclusive of start and end (as with Table).

::
	multi_row_dataframe = df_ts[boolean_array]


Indexing Comparison with AstroPy QTable
######################################

The following are examples of similar functionality using the AstroPy Table: ::
  from astropy.table import Table
  from astropy import units as u
  index = [0.0, 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0] * u.s
  a = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9] * u.m
  b = [10, 11, 12, 13, 14, 15, 16, 17, 18, 19]  * u.K
  c = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j']
  qtable = Table([index, a, b, c], names=('index', 'col_a', 'col_b', 'col_c'), meta={'name': 'first table'})
  qtable.add_index('index')


You can slice using:
::
  row_object = qtable.loc[2.0] = qtable.loc[qtable['index'][2]] # where qtable[2.0], qtable[2.0*u.s] and qtable.loc[2.0*u.s] fail

::
  row_object = qtable.iloc[2] = qtable[2]

::
truncated_qtable = qtable.loc[2.0:4.0] = qtable.loc[qtable['index'][2]:qtable['index'][4]] # where qtable.loc[2.0*u.s:4.0*u.s] fails

::
  truncated_qtable = qtable.loc[1.5:4.5] # values not in index

::
  truncated_qtable = qtable.iloc[2:5] = qtable[2:5]

::
  truncated_qtable = qtable.iloc[2:5:2] = qtable[2:5:2]
	
	
*Note: Table[start:end] = Table.iloc[start:end] is exclusive for end while Table.loc[start:end] is inclusive of start and end (as with DataFrame).*

::
	multi_row_table = qtable[boolean_array]
	
The quantity in a QTable can be acessed using the column's quantity attribute.

::
	quantity = qtable[colname].quantity


Branches and pull requests
--------------------------

N/A


Implementation
--------------

**TODO:**
This section lists the major steps required to implement the APE.  Where
possible, it should be noted where one step is dependent on another, and which
steps may be optionally omitted.  Where it makes sense, each  step should
include a link related pull requests as the implementation progresses.


Backward compatibility
----------------------

This would be new functionality.


Alternatives
------------

Forego the functionality provided by Time and Units and recommend everyone use pandas.


Decision rationale
------------------
