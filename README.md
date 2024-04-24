# Log Continuity - Splunk
Questa è una guida che spiega come è stato implementato il meccanismo di alerting per controllare che gli indici Splunk non presentino un flusso di log anomalo.
Il meccanismo è stato ideato prendendo spunto da questa pagina sulla documentazione ufficiale di Splunk https://docs.splunk.com/Documentation/MLApp/5.4.1/User/Anomalydetectiondashboard

## Prerequisiti
Per far funzionare il meccanismo occorre aver installato sul search head gli addons "Splunk Machine Learning Toolkit (MLTK)" e "Python for Scientific Computing (PSC)"

## Funzionamento in breve
Per ogni indice Splunk monitorato sono definiti i seguenti elementi:

- Un report che allena un modello costruito con l'algoritmo `Random Forest Regressor` utilizzato per predirre il numero di eventi che ci si aspetta di avere in un certo istante temporale
- Un alert che ogni ora calcola il numero di eventi che sono stati inseriti nell'indice in quell'ora e che, applicando il modello, verifica quanto la predizione sia distante dal numero calcolato
- Una lookup table che mantiene uno storico con granularità oraria delle predizioni e dei conteggi di eventi per quell'indice

## Report [MOV][Log Continuity] - Model Training index X
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
