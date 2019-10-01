# Calculating [Welch (2019)](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=3371240) Market Betas in SAS

```sas
%macro welch_market_beta (yr_beg = 2018 , yr_end = 2018 , rw_mon = 60 , delta = 3 , rho = 2 / 252 , output_ds = );

%let dsenames_std_filter = (shrcd in (10 , 11) and exchcd in (1 , 2 , 3));
%let dsf_sample_period = (intnx('mon' , mdy(1,1,&yr_beg.) , - &rw_mon. , 'end') 
                          lt date le "31dec&yr_end."d);

/* data dsf_short; set crsp.dsf; */
/* where &dsf_sample_period.; */
/* keep permno date ret; run; */

proc sql;
create table dsf_short as
select a.permno , b.date , b.ret
from (select * from crsp.dsenames 
      where &dsenames_std_filter.) as a , 
     (select * from crsp.dsf 
      where &dsf_sample_period.) as b
where a.permno eq b.permno and
      a.namedt le b.date le a.nameendt
order by a.permno , b.date
;
quit;

proc datasets lib = work nolist; 
delete &output_ds. _dset0 _dset1 _regest0 _regest1;
run;

%local mdate yy mm;

%do yy = &yr_beg %to &yr_end;
%do mm = 1 %to 12;
/* %do yy = &yr_beg %to &yr_beg; */
/* %do mm = 1 %to 1; */

%let mdate = %sysfunc(mdy(&mm , 1 , &yy));
%let mdate = %sysfunc(intnx(month , &mdate. , 0 , end));
%put ********** Calculating market betas for %sysfunc(putn(&mdate , date9.)) **********;

data _dset0; set dsf_short;
where 0 le intck('mon' , date , &mdate.) lt &rw_mon.;
run;

proc sql;
create table _dset1 as
select a.* , (a.ret - b.rf) as exret , b.mktrf , b.rf
from _dset0 as a
left join ff.factors_daily as b
  on a.date eq b.date
group by a.permno
having count(exret) ge 250
;
quit;
proc sort; by permno date;
data _dset1; set _dset1;
by permno date; 
if first.permno then n = 0; n + 1;
agew = exp(&rho. * n);
exrlo = min((1 - &delta.) * mktrf , (1 + &delta.) * mktrf);
exrhi = max((1 - &delta.) * mktrf , (1 + &delta.) * mktrf);
if not missing(exret) then 
   exret = min(max(exrlo , exret) , exrhi);
drop exrlo exrhi n rf;
run;

proc reg data = _dset1 outest = _regest0 edf noprint;
model exret = mktrf;
by permno; weight agew;
run;

data _regest1;
format permno mdate bMkt;
keep permno mdate bMkt;
set _regest0 (rename = (mktrf = bMkt));
mdate = &mdate.; format mdate date9.;
run;

proc datasets lib = work nolist;
append base = &output_ds. data = _regest1;
delete _dset0 _dset1 _regest0 _regest1;
run;

%end;
%end;
%mend welch_market_beta;

/* %welch_market_beta (output_ds = welch_beta); */
/* %welch_market_beta (yr_beg = 1927 , yr_end = 2018 , rw_mon = 36 , output_ds = welch_beta); */
```
