# Log Continuity - Splunk
Questa è una guida che spiega come è stato implementato il meccanismo di alerting per controllare che gli indici Splunk non presentino un flusso di log anomalo.
Il meccanismo è stato ideato prendendo spunto da questa pagina sulla documentazione ufficiale di Splunk https://docs.splunk.com/Documentation/MLApp/5.4.1/User/Anomalydetectiondashboard

## Prerequisiti
Per far funzionare il meccanismo occorre aver installato sul search head gli addons `Splunk Machine Learning Toolkit (MLTK)` e `Python for Scientific Computing (PSC)`.

## Funzionamento in breve
Per ogni indice Splunk monitorato sono definiti i seguenti elementi:

- Un report che allena un modello costruito con l'algoritmo `Random Forest Regressor` utilizzato per predirre il numero di eventi che ci si aspetta di avere in un certo istante temporale
- Un alert che ogni ora calcola il numero di eventi che sono stati inseriti nell'indice in quell'ora e che, applicando il modello, verifica quanto la predizione sia distante dal numero calcolato
- Una lookup table che mantiene uno storico con granularità oraria delle predizioni e dei conteggi di eventi per quell'indice

## Report - [MOV][Log Continuity] - Model Training index X
```
| tstats `summariesonly` count where index=X groupby _time span=1h
| timechart span=1h sum(count) as count
| fillnull value=0
| eval hour=strftime(_time,"%H")
| eval weekday=strftime(_time,"%a")
| eval weekday_num=case(weekday=="Mon",1,weekday=="Tue",2,weekday=="Wed",3,weekday=="Thu",4,weekday=="Fri",5,weekday=="Sat",6,weekday=="Sun",7)
| eval month_day=strftime(_time,"%m-%d")
| eval is_holiday=case(month_day=="01-01",1,month_day=="01-06",1,month_day=="04-25",1,month_day=="05-01",1,month_day=="06-02",1,month_day=="08-15",1,month_day=="11-01",1,month_day=="12-08",1,month_day=="12-25",1,month_day=="12-26",1,month_day=="12-31",1,1=1,0)
| fields _time count hour weekday_num is_holiday
| fit RandomForestRegressor count from hour is_holiday weekday_num into app:log_continuity_X as predicted
```
- Earliest time: `-30d@d`
- Latest time: `@d`
- Schedule: `0 0 * * *`

La search calcola il numero di eventi avuti sull'indice ogni ora negli ultimi 30 giorni. Viene allenato il modello `log_continuity_X` con l'algoritmo `RandomForestRegressor` usando come parametri l'ora in cui è stato calcolato il numero di eventi, un indicatore che suggerisce se il giorno è festivo oppure no e il giorno della settimana. Viene indicato al modello di predirre il campo `count`.

Il modello si allena ogni giorno sui 30 giorni precedenti, quindi è in grado di adattarsi ad eventuali cambi permanenti nell'andamento del numero di eventi come in figura: 
![alt text](https://github.com/baltornat/log_continuity_docs/blob/main/log_continuity_log_count.png?raw=true)

Il modello risulta inoltre sensibile ai trend giornalieri (giorno/notte/weekend/non-weekend) e alle festività (giorno-festivo/non-festivo).

## Alert - [MOV][Log Continuity] - Populate log continuity lookup index X
```
| makeresults
| eval _time=relative_time(now(), "-1h@h"), count=0, index=X
| append [
   | tstats `summariesonly` count where index=X groupby _time span=1h
]
| timechart span=1h sum(count) as count 
| eval hour=strftime(_time,"%H")
| eval weekday=strftime(_time,"%a")
| eval weekday_num=case(weekday=="Mon",1,weekday=="Tue",2,weekday=="Wed",3,weekday=="Thu",4,weekday=="Fri",5,weekday=="Sat",6,weekday=="Sun",7)
| eval month_day=strftime(_time,"%m-%d")
| eval is_holiday=case(month_day=="01-01",1,month_day=="01-06",1,month_day=="04-25",1,month_day=="05-01",1,month_day=="06-02",1,month_day=="08-15",1,month_day=="11-01",1,month_day=="12-08",1,month_day=="12-25",1,month_day=="12-26",1,month_day=="12-31",1,1=1,0)
| fields _time count hour weekday_num is_holiday
| apply app:log_continuity_X as predicted
| eval error=count-predicted
| outputlookup append=t log_continuity_X.csv
| append [| inputlookup log_continuity_X.csv]
| eval error_sq=error*error
| eventstats avg(error_sq) as mean_error 
| eventstats stdev(error_sq) as sd_error 
| eval z_score=if(sd_error!=0, (error_sq-mean_error)/sd_error, 0)  
| sort 0 + _time 
| tail 1
| eval z_score=if(z_score>0, z_score, 0)
| eval index="X"
| table _time index count predicted z_score
```
- Earliest time: `-1h@h`
- Latest time: `@h`
- Schedule: `1 * * * *`
- Trigger condition: `search z_score>6`
- Trigger action: `Send email`

L'alert ha molteplici funzioni:
1. Calcolare ogni ora il numero di eventi scritti nell'indice (in quell'ora) e applicare il modello per predirre il numero di eventi atteso
2. Appendere il risultato ottenuto alla lookup che mantiene lo storico orario `log_continuity_X.csv`
3. Leggere l'intero contenuto della lookup e calcolare le metriche per definire se il numero di eventi scritti nell'indice in quell'ora è distante oppure no dal valore predetto. Se il numero di eventi è distante 6 o più deviazioni standard dal predetto (in positivo o in negativo) allora deve essere inviata una mail di segnalazione

Ricapitolando:
![alt text](https://github.com/baltornat/log_continuity_docs/blob/main/log_continuity.png?raw=true)

## Lookup - log_continuity_X.csv



