# Grupp 4 Logic App övervakning av hemsida

### Vad appen ska göra
Vi har fått i uppgift att välja ett projekt som vi ska göra i grupp. Vi har valt att övervaka en hemsida med hjälpa av Logic App i Microsoft Azure som analyserar bland annat svarstid, om sidan är tillgänglig eller inte m.m.<br>
Vi har gjort en enkel app som läser av sidan Faultnode.se var 15 minut. Faultnode.se har bara ett syfte, det är att man ska kunna kontrolera sidan och få förutsägbara fel, tex lång svarstid, HTTP koder som 500, 503 och att sidan inte är åtkomlig. <br>
Den information som appen fångar upp från Faultnode.se kommer sedan att visas i en SharePoint Site.

<details>
<summary>Olika Sharepoint Actions som används i vår Logic App</summary>
<h3><strong>Compose</strong></h3>
Compose – används för att hålla eller bearbeta ett värde under en körning, till exempel ett tidsvärde från utcNow() eller en uträkning. I vår Logic App använder vi bland annat Compose för StartTime, EndTime, Response Time och HTTPStatus.<br>
<img width="470" height="185" alt="image" src="https://github.com/user-attachments/assets/62dd90b7-0121-4e74-8611-cac16eaad2c8" />

<h3><strong>Condition</strong></h3>
Condition – används för att kontrollera om ett villkor är sant eller falskt. Beroende på resultatet går Logic Appen vidare i antingen True- eller False-grenen. Hos vi använder det exempelvis för att kontrollera HTTP-status och om sidan är DOWN.<Br>
<img width="479" height="288" alt="image" src="https://github.com/user-attachments/assets/f60b2d89-f540-408e-abb1-393cddfe43e9" />

<h3><strong>Get item</strong></h3>
Get item – hämtar en specifik post från en SharePoint-lista. Vi använder GetMonitorState för att läsa information som ska finnas kvar mellan olika körningar av Logic Appen.<br>
Här krävs ett ID för att Get Item ska veta vilken post den ska använda.<br>
<img width="477" height="334" alt="image" src="https://github.com/user-attachments/assets/4685a845-4275-4ddd-a2ef-55e345e0fa01" />

<h3><strong>Update item</strong></h3>
Update item – uppdaterar en befintlig post i en SharePoint-lista. Vi använder den bland annat för att sätta FirstFailureTime och uppdatera MonitorState.<br>
<img width="469" height="625" alt="image" src="https://github.com/user-attachments/assets/bfbc5084-de6b-44e4-9866-fa2d9f7d2f8d" />






</details>


## Appens uppbyggnad
Vår app är uppbyggd i tre delar.<br>
*Del 1* <br>
Första delen övervakar den valda hemsidan och lagrar information.<br>
<details>
 <summary>Bild 1 </summary>
<img width="352" height="689" alt="image" src="https://github.com/user-attachments/assets/fa3d9178-dd51-4750-a628-335a2065ce9c" />
</details>


*Del 2* <br>
Andra delen bearbetar informationen, sätter **WebsiteStatus** till ett av tre värden beroende på status på hemsidan som övervakas.<br>
<details>
 <summary>Bild 2 </summary>
<img width="1221" height="930" alt="image" src="https://github.com/user-attachments/assets/63401a56-53d5-4da0-a07e-4236381425de" /> <br>
</details>

  Tredje delen använder variabeln **WebsiteStatus** för att bestämma hur **MonitorState** ska uppdateras.
  <details>
 <summary>Bild 3 </summary>
<img width="1311" height="1055" alt="image" src="https://github.com/user-attachments/assets/6e4fc9d2-8547-46ee-b0e2-97545c1da6de" />
</details>

## Hur appen fungerar, första blocket
Appen använder en **Recurrence** som trigger, den upprepas var 15:e minut och inställd på UTC+01:00<br>
Det första som händer efter att triggern startar en ny körning är **Compose** som vi har bytt till namnet **StartTime**.<br>
**StartTime** kör uttrycket **utcNow()** och sparar resultatet som sin output. Nästa del i kedjan är **HTTP** denna har inte fått ett eget namn, HTTP har som uppgift att anropa sidan: https://faultnode.se/ vi gör det med metoden **GET**. <br>
Den skickar helt enkelt en HTTP-förfrågan och sidans server svarar med bland annat en HTTP-statuskod till exempel 200, 500 eller 503. **HTTP** sparar svaret sedan som sin output. Nästa del i kedjan kommer inte köra innan **HTTP** har fått ett svar från servern. Tack vare detta kan vi räkna ut hur lång svarstid
servern har men ytterligare en **Compose** som vi döper till **EndTime**. EndTime använder också uttrycket **utcNow()** och gör det direkt efter **HTTP** är klar. Här kan det uppstå problem om **HTTP** inte körs klart och får ett **Has failed** eller **Has timed out**,
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
Andra blocket börjar med ett **Condition** som heter **IfStatusCode500or503** det kollar helt enkelt om outputen från HTTPStatus är 500/503 eller om den inte är det, se bild 2.<br> 
<img width="500" height="380" alt="image" src="https://github.com/user-attachments/assets/a93effca-b5b5-4ba1-a83b-4571b39e9516" /> <br>
Här måste vi använda **Condition expression** "or" eftersom att det är två olika värden vi vill kontrollera.

Om **HTTPStatus** är 500 eller 503 är resultatet **True** och sätter vår variabel **WebsiteStatus** till **DOWN**. Om **WebsiteStatus** är **UP** betyder det att vårt resultat är **False** och vi kommer gå vidare till vårt andra condition i detta block. <br>
I **IfSlowerThan3000MS** kontrollerar vi om output från **Response Time** är längre än 3000MS är den det får vi **True** och vi sätter våran variabel **WebsiteStatus** till **SLOW**, är **Response Time** inte längre än 3000MS sätter vi vår variabel **WebsiteStatus** till **UP**. <br>
Ett av dessa värden kommer vi senare skicka till en SharePoint List beroende på vilket resultat vi får. <br>

Det sista vi har i andra blocket är en **Get item** detta är en Sharepoint-action som hämtar en specifik rad från en sharepoint list, vår **Get item** har vi döpt till **GetMonitorState**. Vi behöver information från denna lista eftersom varje gång **Recurrence** körs, vår trigger, börjar hela vår Logic App om från
noll. Ta vår Compose **StartTime** från första blocket som exempel, där har vi valt Inputs som **utcNow()** alltså vad tiden är precis när **StartTime** körs, denna input sparas igenom hela körningen av vår Logic App, men så fort **Recurrence** körs igenom och vår app startas på nytt har dessa värden försvunnit och
**StartTime** kommer ta ett nytt input värde från **utcNow()**. Därför har vi gjort Sharepoint site **MonitorState**, den har bara 4 kolumner, **Website**, **FirstFailureTime** **AlertActive** och **ID**. Här kommer värdena som vi behöver finnas kvar även när vår Logic App kör **Recurrence**. Denna information kommer vi behöva i block tre.

## Tredje blocket
Det första vi har i block tre är ett **Condition** denna är döpt **IsWebSiteDown** den kollar om **WebSiteStatus** = **DOWN**. **IsWebSiteDown** leder till ytterligare till två **Conditions**, **IsFirstFailureTimeEmty** och **WasAlertActive**, se bild 3. <br>
Vi säger att vi börjar från ett stadie där Faultnode.se fungerar som det ska, sidan ligger inte nere. **MonitorState** kommer då se ut så här: <br>
<img width="954" height="191" alt="image" src="https://github.com/user-attachments/assets/d3e375e0-eb8a-4e16-b1ec-6640b99f6d67" /> <br>


### Första körningen efter Faultnode.se gått ner
Men plötsligt svarar inte Faultnode.se, efter ca 15 min körs **Recurrence** igen och Logic App startar om. Denna gång kommer **WebsiteStatus** att sättas till **DOWN** och **IsWebSiteDown** i tredje blocket kollar: är **WebsiteStatus** = **DOWN**. I detta fall blir det **True**. <br>
Logic App går då vidare till **True** där har vi placerat ytterligare ett **Condition**: **IsFirstFailureTimeEmpty**.<br> 
**IsFirstFailureTimeEmpty** kontrollerar om värdet **FirstFailureTime** i våran lista, se bild MonitorState, är tom. I vårt fall är den tom och resultatet blir **True**. Logic app går vidare till **True** och kör en Sharepoint-action **Update Item** som vi har döpt till: **SetFirstFailureTime**. <br>
**SetFirstFailureTime** uppdaterar listan **MonitorState** och sätter **FirstFailureTime** till den aktuella tiden med **utcNow()**. <br>

<img width="963" height="155" alt="image" src="https://github.com/user-attachments/assets/895869d2-29f1-4ab8-bf3f-9f31b9101a0b" />

Det sista Logic App gör nu är en till Sharepoint-action, **Create Item** vår heter: **CreateSiteLogsEntry**, den skickar informationen till vår Sharepoint lista **Site logs** och fyller i: **HTTP Status**, **TimeStamp**, **ResponseTimeMS**, **WebsiteStatus** och **Website**.  Efter det är vår Logic App klar för denna körningen, den gör inget mer förrän **Recurrence** körs igen efter ca 15min. <br>

### Andra körningen efter att Faultnode.se gått ner
Låt oss säga att Faultnode.se fortfarande ligger nere. Vår Logic App kommer gå igenom samma sak igen. Den kommer gå igenom start time, HTTP response time, sätta en variabel. Eftersom att Faultnode.se fortfarande ligger nere så kommer den sätta website status till **DOWN**. Vi går vidare och vår **MonitorState** lista är uppdaterad. Förra körningen la till ett värde i **FirstFailureTime**. Och vår variabel **WebsiteStatus** är **DOWN**, **IsWebsiteDown** kollar om **WebsiteStatus** = **DOWN**. Vi går vidare till **True**. Nästa **IsFirstFailureTimeEmpty** det är den inte i vår Sharepoint list i kolumn **FirstFailureTime** har vi ett värde, Logic App går vidare till **False**.<br>
Nästa **Condition** vi kommer till är **Has30MinutesPassed** den använder **addMinutes()** den tar helt enkelt värdet i **FirstFailureTime**, lägger på 30 minuter, sedan jämförs det värdet med ett nytt **utcNow()**. Om **utcNow()**, är större eller lika med **FirstFailureTime** + 30 minuter kommer Logic App gå vidare till **True**, i vårt fall är det **False**
eftersom detta är första körningen efter att Faultnode.se gick ner. Logic App går till **False** och inget mer händer här. Efter det lägger **CreateSiteLogsEntry** en ny rad i **Site logs** och körningen avslutas.<br>

### Tredje körninge efter Faultnode.se gått ner
Efter 15 minuter körs **Recurrence** igen, Logic App går igenom alla **Conditions**. Denna gång vid **Has30MinutesPassed** kommer den vara **True**, vår logic app kör var 15:e minut och detta är tredje körningen alltså har minst 30 minuter gått sedan Faultnode.se gick ner. Vi går vidare till **IsAlertInactive**, den kontrollerar kolumn **AlertActive** i vår Sharepoint list **MonitorState**. Just nu har **AlertActive** **No** i sin kolumn, se tidigare bild. Alltså går vår **Condition** **IsAlertInactive** vidare till **True**, här skickar vi ett e-mail till vald e-mail adress med information om att Faultnode.se har varit nere i 30 minuter, vi uppdaterar också vår sharepoint list **MonitorState** så **AlertActive** står på **Yes**, **CreateSiteLogsEntry** körs och appen avslutas.<br> 
Efter den tredje körningen av vår Logic App kommer **MonitorState** se ut så här, se bild nedanför. **IsAlertInactive** kontrollerar om **AlertActive** står på **Yes** eller **No** vid varje körning, om vi inte haft det hade vi fått ett e-mail utskickat till oss var 15 minut så länge sidan ligger nere. Vi undviker detta genom att sätta **AlertActive** till **Yes**
så varje gång **IsAlertInactive** körs och Faultnode.se inte har kommit online igen går Logic App till **False** och inget e-mail skickas.

<img width="968" height="180" alt="image" src="https://github.com/user-attachments/assets/47d5ce8a-f97f-43bc-9497-823b0a8504f8" />

### Faultnode.se är online igen
När Faultnode.se väl är online igen och vår Logic App körs igen kommer den att registrera Faultnode.se som **Up** igen, **WebsiteStatus** är **UP**. **IsWebSiteDown** kollar om **WebsiteStatus** är **DOWN** det är den inte, och Logic App går vidare till **False**. **WasAlertActive** kontrollerar om **AlertActive** står på **Yes** och det gör den, Logic App går vidare till **True**, skickar ett e-mail till vald e-mail adress och kör **ResetMonitorState**. <br>
**ResetMonitorState** nollställer **MonitorState** till, se bild. Nu undviker vi också att skicka ett e-mail var 15:e minut, varje gång Logic App kör och Faultnode.se är online kommer **WasAlertActive** gå till **False** eftersom **AlertActive** står på **No**.
<img width="856" height="160" alt="image" src="https://github.com/user-attachments/assets/b04965e4-020d-40b0-a7e2-ed3ef2582876" />





















