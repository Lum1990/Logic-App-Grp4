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

## Hur appen fungerar
Appen använder en Recurrence som trigger, den upprepas var 15´e minut och inställd på UTC+01:00<br>
Det första som händer efter att triggern startar en ny körning är **Compose** som vi har bytt till namnet **StartTime**.<br>
StartTime kör uttrycket **utcNow()** och sparar resultatet som sin output. Nästa del i kedjan är **HTTP** denna har inte fått ett eget namne, HTTP har som uppgift att anropa sidan: https://faultnode.se/ vi gör det med metoden **GET**. <br>
Den skickar helt enkelt en http-förfrågan och sidans server svarar med en http-statuskod till exempel 200, 500 eller 503. Nästa del i kedjan kommer inte köra innan HTTP har fått ett svar från serven. Tack vare detta kan vi räkna ut hur lång svarstid
serven har men yttligare en **Compose** som vi döper till **EndTime**. EndTime använder också uttrycket utcNow() och gör det direkt efter HTTP är klar. Här kan det uppstå problem om HTTP inte körs klart och får ett **Has failed** eller **Has timed out**,
det kan vi åtgärda i settings i EndTime och låta den köra även om HTTP får något av dessa fel. <img width="472" height="379" alt="image" src="https://github.com/user-attachments/assets/950ffffb-b26a-4493-8753-e5867165dd59" />

För att räkna ut hur lång tid anropet tog använder vi ticks(), det gör om StartTime och EndTime till heltal som vi kan subtrahera med varandra och sedan dividera med 10 000 för att räkna ut antalet millisekunder anropet tog.<br>
Detta gör vi en ny **Compose** som heter **Response Time**.
````
div(
  sub(
    ticks(outputs('EndTime')),
    ticks(outputs('StartTime'))
  ),
  10000
)
````





