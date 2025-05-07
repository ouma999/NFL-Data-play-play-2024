# NFL-Data-play-play-2024

/*Import the dataset into sas environment */
PROC IMPORT datafile="/home/u64133399/DSCI_507/PG2/data/play_by_play_2024.csv"
    out=mydata
    dbms=csv
    replace;
    getnames=yes;
RUN;
/*create a temporary library and Filter the dataset where condition down=4.*/
Data work.fourth_down_clean;
    set mydata;
    where down = 4;
run;
/*viewing table sand column attributes*/
proc contents data= fourth_down_clean;
run

/* Create a decision for 4th down play types */
DATA work.fourth_down_model;
    SET work.fourth_down_clean;
    IF play_type IN ('run', 'pass') THEN go_for_it = 1;
    ELSE IF play_type IN ('punt', 'field_goal') THEN go_for_it = 0;
    ELSE go_for_it = .;
RUN;
/*view the first 10 rows and observation of the listed variables*/

proc print data=fourth_down_model (obs=10);
    
    var game_id 
        qtr 
        game_seconds_remaining 
        down 
        ydstogo 
        yardline_100 
        posteam 
        defteam 
        score_differential 
        play_type_nfl 
        epa 
        yards_gained 
        drive_play_count 
        go_for_it;  
run;





/*Select relevant variables for your analysis*/

DATA work.fourth_down_decision_model;
    SET work.fourth_down_model;
/* Keep only relevant variables */
    KEEP game_id qtr game_seconds_remaining down ydstogo yardline_100 
         posteam defteam score_differential play_type_nfl epa yards_gained drive_play_count go_for_it;       
RUN;

/*Summarize the distribution of key variables using frequency tables.*/
PROC FREQ data=work.fourth_down_decision_model;   
RUN;
/*Perform a two-way cross-tabulation between the quarter of the game and the
 binary decision variable (go-for-it vs kick).*/

PROC FREQ  data=work.fourth_down_decision_model;
    tables qtr*go_for_it / 
        norow        
        nocol       /*suppress*/
        nopercent;

RUN;

/* Chi-Square Test of Independence to determine whether the quarter
 is associated with the 4th down decision.*/

ods graphics on;

PROC FREQ Data=work.fourth_down_decision_model;
    tables qtr*go_for_it / chisq expected nocol norow nopercent plots=freqplot;
RUN;

ods graphics off;


/*variable that groups quarters*/

DATA work.fourth_down_decision_model;
    set work.fourth_down_decision_model;

/* if elfe statement that Create a new variable 'half'*/
    if qtr in (1, 2) then half = "Early";
    else if qtr in (3, 4,5) then half = "Late";
RUN;

/*//Then run an odds ratio analysis:/*/
PROC FREQ Data=work.fourth_down_decision_model;
    tables half*go_for_it / chisq or;
RUN;


/*PREDICTIVE MODELLING*/

/*Building a log regression*/
/*If the p-value is < 0.05, reject the null: the model has predictive power.*/
ods graphics on;
proc logistic data=work.fourth_down_decision_model descending;

    class qtr posteam defteam / param=ref ref=first;

    model go_for_it = 
        qtr 
        game_seconds_remaining 
        ydstogo 
        yardline_100 
        posteam 
        defteam 
        score_differential 
        epa 
        yards_gained 
        drive_play_count;

run;
ods graphics off;

/*Model Refinement
Perform a stepwise variable selection procedure to refine your model.
*/
ods graphics on;
proc logistic data=work.fourth_down_decision_model descending;

    class qtr posteam defteam / param=ref ref=first;

    model go_for_it = 
        qtr 
        game_seconds_remaining 
        ydstogo 
        yardline_100 
        posteam 
        defteam 
        score_differential 
        epa 
        yards_gained 
        drive_play_count
    / selection=stepwise slentry=0.05 slstay=0.05;

run;

ods graphics off;
