# Grupp 4 Logic app övervakning av hemsida

### Vad appen ska göra
Vi har fått i uppgift att välja ett projekt som vi ska göra i grupp. Vi har valt att övervaka en hemsida men hjälpa av Logic App i Microsoft Azure som analyserar bland annat svarstid, om sidan är tillgänglig eller inte m.m.<br>
Vi har gjort en enkel app som läser av sidan Faultnode.se var 15 minut. Faultnode.se har bara ett syfte, det är att man ska kunda kontrolera sidan och få förutsägbara fel, tex lång svarstid, HTTP koder som 500, 503 och att sidan inte är åtkommlig. <br>
Den information som appen fångar upp från Faultnode.se kommer sedan att visas i en SharePoint Site.

## Appens uppbyggnad
Vår app är uppbyggd i två huvuddela.<br>
*Del 1* <br>
Första delen övervakar den valda hemsaidan och lagrar information.<br>
<details>
 <summary>Bild 1 </summary>
<img width="352" height="689" alt="image" src="https://github.com/user-attachments/assets/fa3d9178-dd51-4750-a628-335a2065ce9c" />
</details>


*Del 2* <br>
Andra delen bearbetar informationen, sätter variabler och skickar informationen till SharePoint.<br>
<details>
 <summary>Bild 2 </summary>
<img width="1221" height="930" alt="image" src="https://github.com/user-attachments/assets/63401a56-53d5-4da0-a07e-4236381425de" /> <br>
</details>
  
  <details>
 <summary>Bild 3 </summary>
<img width="1311" height="1055" alt="image" src="https://github.com/user-attachments/assets/6e4fc9d2-8547-46ee-b0e2-97545c1da6de" />
</details>

## Hur appen fungerar, första blocket
Appen använder en **Recurrence** som trigger, den upprepas var 15´e minut och inställd på UTC+01:00<br>
Det första som händer efter att triggern startar en ny körning är **Compose** som vi har bytt till namnet **StartTime**.<br>
**StartTime** kör uttrycket **utcNow()** och sparar resultatet som sin output. Nästa del i kedjan är **HTTP** denna har inte fått ett eget namn, HTTP har som uppgift att anropa sidan: https://faultnode.se/ vi gör det med metoden **GET**. <br>
Den skickar helt enkelt en HTTP-förfrågan och sidans server svarar med bland annat en HTTP-statuskod till exempel 200, 500 eller 503. **HTTP** sparar svaret sedan som sin output. Nästa del i kedjan kommer inte köra innan **HTTP** har fått ett svar från serven. Tack vare detta kan vi räkna ut hur lång svarstid
serven har men yttligare en **Compose** som vi döper till **EndTime**. EndTime använder också uttrycket **utcNow()** och gör det direkt efter **HTTP** är klar. Här kan det uppstå problem om **HTTP** inte körs klart och får ett **Has failed** eller **Has timed out**,
det kan vi åtgärda i settings i **EndTime** och låta **EndTime** köra även om **HTTP** får något av dessa fel.<br> 

<img width="472" height="379" alt="image" src="https://github.com/user-attachments/assets/950ffffb-b26a-4493-8753-e5867165dd59" />

För att räkna ut hur lång tid anropet tog använder vi **ticks()**, det gör om **StartTime** och **EndTime** till heltal och vi kan subtrahera **StartTime** från **EndTime** och sedan dividera med 10 000 för att räkna ut antalet millisekunder anropet tog.<br>
Detta gör vi i en ny **Compose** som får namnet **Response Time** här använder vi **div()**.
````
div(
  sub(
    ticks(outputs('EndTime')),
    ticks(outputs('StartTime'))
  ),
  10000
)
````

Det sista som sker i detta block, se bild 1, är en sista **Compose**, **HTTPStatus** som hämtar **statusCode** från **HTTP** och sparar det som sin egen output. <br>
Vi skapar även en **Initialize variables**, som inte heller fått ett eget namn, där skapar vi variabeln **WebsiteStatus** som är av typen String, just nu är den tom, vi ger den ett värde i nästa steg.

## Andra blocket
Andra blocket börjar med ett **Condition** som heter **IfStatusCode500or503** det kollar helt enkelt om outputen från HTTPStatus är 500/503 eller om den inte är det.<br> 
<img width="500" height="380" alt="image" src="https://github.com/user-attachments/assets/a93effca-b5b5-4ba1-a83b-4571b39e9516" /> <br>
Här måste vi använd **Condition expression** "or" efter som att det är två olika värden vi vill kontrolera.

Om **HTTPStatus** är 500 eller 503 är resultatet **True** och sätter vår variabel **WebsiteStatus** till **DOWN**. Om **HTTPStatus** är **UP** betyder det att vårt resultat är **False** och vi kommer gå vidare till vårt andra condition i detta block. <br>
I **IfSlowerThan3000MS** kontrolerar vi om output från **Response Time** är längre än 3000MS är den det får vi **True** och vi sätter våran variabel **WebsiteStatus** till **SLOW**, är **Response Time** inte längre än 3000MS sätter vi vår variabel **WebsiteStatus** till **UP** <br>
Ett av dessa värden kommer vi senare skicka till en SharePoint List beroende på vilket resultat vi får.






