# Assignment 1 - Benchmarking

In this assignment you will be going over some sample code, and adjusting it for another firm. 

## Before working through this document, please go over the materials on [https://github.com/JoostImpink/SAS-bootcamp](https://github.com/JoostImpink/SAS-bootcamp)

Review the following code (scroll down for the assignment requirements, which are included after an explanation of the code to review):

```SAS
/*  Example of how to use SAS to retrieve data from WRDS
    Computing market-to-book ratio for years 2000-, and benchmark it against
    other firms in the industry */

/* this piece of code makes a connection of your SAS instance with WRDS remote server */
%let wrds = wrds-cloud.wharton.upenn.edu 4016;options comamid = TCP remote=WRDS;
signon username=_prompt_;


/* Let's get the MTB data for Google, notice how we compute it straight in the query */
rsubmit;
proc sql;
	/* create a table and name it 'myData' */
	create table myData as
		/* which variables to select: company name, fiscal year and compute market to book */
		select conm, fyear, (csho * prcc_f / ceq) as mtb
		/* where to get it from: compustat fundamental annual */
		from comp.funda
		/* filter: just get Google */
		where TIC eq "GOOGL"
		/* this is some boilerplate filtering (gets rid of doubles) */
		and indfmt='INDL' and datafmt='STD' and popsrc='D' and consol='C';

quit;
proc print;run;
endrsubmit;


/* How about a benchmark: median firm in the industry (SIC: 7370) 
	7370: SERVICES-COMPUTER PROGRAMMING, DATA PROCESSING, ETC., see https://www.sec.gov/info/edgar/siccodes.htm */
rsubmit;
proc sql;
	/* create a table and name it 'myData2' */
	create table myData2 as
		/* which variables to select: fiscal year and compute market to book */
		select fyear, count(*) as numFirms, median(mtb) as median_mtb
		/* where to get it from: a subquery */
		from (
			select fyear, (csho * prcc_f / ceq) as mtb from comp.funda
			/* filter: get all firms in industry 7370 after 2000 */
			where SICH eq 7370 and fyear > 2000
			/* this is some boilerplate filtering (gets rid of doubles) */
			and indfmt='INDL' and datafmt='STD' and popsrc='D' and consol='C'
			)
		/* compute it for each year => GROUP BY */
		group by fyear;

quit;
proc print;run;

/* lets combine the two tables (match on year) */
proc sql;
	create table myData3 as 
	/* from table a (myData) get fyear and mtb (rename as mtb_google) */
	select a.fyear, a.mtb as mtb_google, 
	/* from table b (myData2) get everything ('*') */
	b.* 
	from myData a, myData2 b 
	/* join on fiscal year (we want to match Google's mtb to the industry median for each year)*/
	where a.fyear = b.fyear;
quit;
proc print;run;
endrsubmit;
```

## Some explanations

### SQL example

The following query creates a new dataset (named 'myData'). It takes `comp.funda` as a starting point and filters on `TIC eq "GOOGL"`, effectely only getting Google data (`eq` means equals). There is also some boilerplate filters (that are always needed when getting data from Compustat Fundamental Annual or Quarterly) `and indfmt='INDL' and datafmt='STD' and popsrc='D' and consol='C'`. This is to prevent double/wrong records.


```SAS
/* Let's get the MTB data for Google, notice how we compute it straight in the query */
rsubmit;
proc sql;
	/* create a table and name it 'myData' */
	create table myData as
		/* which variables to select: company name, fiscal year and compute market to book */
		select conm, fyear, (csho * prcc_f / ceq) as mtb
		/* where to get it from: compustat fundamental annual */
		from comp.funda
		/* filter: just get Google */
		where TIC eq "GOOGL"
		/* this is some boilerplate filtering (gets rid of doubles) */
		and indfmt='INDL' and datafmt='STD' and popsrc='D' and consol='C';

quit;
proc print;run;
endrsubmit;
```

The `select conm, fyear, (csho * prcc_f / ceq) as mtb` tells SAS what we want from 'comp.funda'. 'conm' is company name, 'chso' is common shares outstanding, 'prcc_f' is end of fiscal year stock price, 'ceq' is book equity.
Notice how we can make new variables 'as mtb' names the newly created variable. Tutorial [2. Finding your way on WRDS](2_using_wrds_website) covers how you can find out which variable names are used in the tables.

### Proc print

Outputting the contents of the last dataset used can be done with 'proc print':

```SAS
proc print;run;
```

> Note: procedures typically end with a 'run;' or 'quit;'' statement.

If a dataset is large you can print a subset with 'obs=..', for example:

```SAS
proc print data=myData (obs = 10);run;
```

This specifies which table to print (in this case 'myData'), and prints the first 10 observations.


### Subquery

The next code block contains a subquery. In the `from` part (which dataset to read), it doesn't specify a dataset. Instead, there is another 'select' (the subquery). The subquery takes 'funda' as its input ('from comp.funda'):


```SAS 
proc sql;
	/* create a table and name it 'myData2' */
	create table myData2 as
		/* which variables to select: fiscal year and compute market to book */
		select fyear, count(*) as numFirms, median(mtb) as median_mtb
		/* where to get it from: a subquery */
		from (
			select fyear, (csho * prcc_f / ceq) as mtb from comp.funda
			/* filter: get all firms in industry 7370 after 2000 */
			where SICH eq 7370 and fyear > 2000
			/* this is some boilerplate filtering (gets rid of doubles) */
			and indfmt='INDL' and datafmt='STD' and popsrc='D' and consol='C'
			)
		/* compute it for each year => GROUP BY */
		group by fyear;

quit;
```

Focusing on the subquery:

```SAS
from (
			select fyear, (csho * prcc_f / ceq) as mtb from comp.funda
			/* filter: get all firms in industry 7370 after 2000 */
			where SICH eq 7370 and fyear > 2000
			/* this is some boilerplate filtering (gets rid of doubles) */
			and indfmt='INDL' and datafmt='STD' and popsrc='D' and consol='C'
			)
```

This creates a dataset with 'fyear' and 'mtb' for all firms in industry 7370 (Google's industry) for years after 2000. The boilerplate filter (`and indfmt='INDL' and datafmt='STD' and popsrc='D' and consol='C'`) needs to be included as well.

The 'outside' query takes this as its input and for each year computes the number of observations (`count(*) as numFirms`) and the median market to book (`median(mtb) as median_mtb`):

```SAS
select fyear, count(*) as numFirms, median(mtb) as median_mtb
		/* where to get it from: a subquery */
		... subquery ...
		/* compute it for each year => GROUP BY */
		group by fyear;
```		

The `group by fyear` makes this happen. The `group by` (or sometimes just `by`) is a powerful way to repeat certain operations on groups (by firm, by year, by industry, etc). In this case, it computes a median value for each year.

### Join

The last code is a join of datasets myData (which holds fyear and Google's mtb) and myData2 (which holds fyear, numFirms and the industry median mtb).

Note how these two tables are named in the 'from' (`from myData a, myData2 b`). The `a` and `b` means that these two tables may be referenced to as 'a' and 'b' (in the 'select' and 'where' parts).

In this case we want 'fyear' and 'mtb' from 'a', and all variables on 'b' (done with `b.*`). 

A join typically has a criterion on how the join should work. In this case both tables have one record for each year, and we want to combine the data by year. That is, for each year we want to have Google's mtb and the industry median mtb. 
Hence we require that fyear needs to match (`where a.fyear = b.fyear`). 

```SAS
/* lets combine the two tables (match on year) */
proc sql;
	create table myData3 as 
	/* from table a (myData) get fyear and mtb (rename as mtb_google) */
	select a.fyear, a.mtb as mtb_google, 
	/* from table b (myData2) get everything ('*') */
	b.* 
	from myData a, myData2 b 
	/* join on fiscal year (we want to match Google's mtb to the industry median for each year)*/
	where a.fyear = b.fyear;
quit;
```

## Required

1. Select a non-financial firm that is in the [Dow Jones 30 index](http://money.cnn.com/data/dow30/)

2. Find the gvkey for this firm, and the industry it is in using [the WRDS Search Company](https://wrds-web.wharton.upenn.edu/wrds/code_search/) form. (Note: you can search for the ticker with -- for example -- `ticker:INTC` (Intel), look at columns `gvkey` and `sic`)

3.  Compute the following financial ratios over the last 10 years (for assets, use end-of-year numbers):
	- Return on assets (ROA, net income divided by assets)
	- Return on sales (ROS, net income divided by sales)
	- Asset turnover (AT, sales divided by assets)

4. Compute the ratios for your firm and take the median value of these ratios for other firms in your firm's industry. Tabulate the ratios by year: your company vs industry median.

#### Hand in your SAS code and generated tables (copy/paste to Word; doesn't have to be pretty)

## Alternative way to find the gvkey and industry code (SIC)

As an alternative to the firm search form, you can use the following code snippet which selects the ticker symbol and industry code from Funda:  

```SAS
rsubmit;
/* Keep relevant variables -- SICH is the industry code */
data mynames2 (keep = gvkey fyear tic conm sich);
set comp.funda;
/* tic is the variable name for ticker symbol */
where tic eq "INTC"; 
/* prevent double records */
if indfmt='INDL' and datafmt='STD' and popsrc='D' and consol='C' ;
run;
proc download data=mynames2 out=mynames2;run;
endrsubmit;
```

## Notes

1. These are the variables' names in Compustat Funda (see WRDS website for comprehensive list of variables in Compustat Funda, click 'Variables' on https://wrds-web.wharton.upenn.edu/wrds/ds/compd/funda/index.cfm?navId=83):
	- Assets: AT
	- Net income: NI
	- Sales: SALE

2. You can use either SAS in UF Apps or SAS Studio. If you use SAS Studio you don't need rsubmit.

3. Do not use the WRDS web form to manually download the data. Also don't use Excel. The purpose is to do both data retrieval and processing *programmatically* with SAS.

4. Please put comments in your code

