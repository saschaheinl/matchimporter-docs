# DBLV.MatchImporter.API Dokumentation
Die DBLV.MatchImporter.API ist eine ASP.NET API, die in C# geschrieben wurde, um Spielberichte und den Spielverlauf nach Beendigung eines Badminton-Bundesliga-Spieltages sichern zu können. Voraussetzung ist, dass das [Badminton Umpire Panel](https://github.com/phihag/bup) (BUP) eingesetzt wurde. Dabei ist es unwichtig, ob über Courtspot, Badmintonticker oder andere Tickersysteme angebunden wurde. Die Struktur sieht nur vor, dass BUP eingesetzt wurde. 

## Eingesetzte Technologien
Die verwendeten Technolgien lassen sich unterteilen in direkt im Code verwendete Frameworks und in die Infrastruktur, die die Applikation zur Ausführung benötigt. 

### Frameworks und Pattern
Als ASP.NET-Anwendung verwendet die DBLV.MatchImporter.API selbstredend das .NET Framework sowie die dazugehörigen Pakete. Darüber hinaus ist die Applikation nach dem Clean Architecture Pattern aufgebaut. Das sei hier nur kurz beschrieben, genauere Informationen zu diesem lassen sich beispielsweise [hier](https://lewisjohnbaxter.medium.com/understanding-the-clean-architecture-pattern-in-software-development-7a26a494419d#:~:text=The%20Clean%20Architecture%20pattern%20is%20a%20powerful%20approach%20to%20software,to%20changing%20requirements%20and%20technologies.) finden.

Darüber hinaus wird mit dem Mediator-Pattern gearbeitet. Das bedeutet, dass nicht in Services, sondern in Use Cases gearbeitet wird. Diese werden auch nicht direkt aufgerufen, sondern durch den Mediator, dieser wird durch Dependency Injection zur Laufzeit instanziiert. Anschließend wird ein Request an den Mediator geschickt, um den entsprechenden Code aufzurufen. Ein Request erbt von `IRequest<T>`, wobei T eine Query zum Abrufen oder ein Command zum Speichern oder Ändern von Informationen ist. Beide sind im Projekt `DBLV.MatchImporter.API.Core` gespeichert. Im gleichnamigen UseCase (zum Beispiel `GetContactByIdQuery` und `GetContactByIdUseCase`) ist dann der entsprechende Code zu finden, der die entsprechende Aktion ausführt.

Entity Framework wird eingesetzt, um die Calls zur SQL-Datenbank zu vereinfachen. Hier wurde nach dem Prinzip "Code First" gearbeitet, wobei die entsprechende Kontext-Klasse vorgibt, wie die entsprechende Struktur in der Datenbank aussieht. Will man also weitere Spalten in der Tabelle haben, muss die Klasse verändert und eine Migration durchgeführt werden. Weitere Infortmationen zum Thema finden sich auf [Microsoft Learn](https://learn.microsoft.com/en-us/ef/ef6/modeling/code-first/migrations/)

### Infrastruktur
Der Code liegt zurzeit in einem [Repo](https://github.com/hnlvc/Dblv.MatchImporter.API) auf GitHub. Dort ist eine CI/CD Pipeline konfiguriert, die das Projekt kompiliert und auf Amazon Web Services (AWS) veröffentlicht. Dort läuft die API in einem Elastic Cloud Service Container. Dieser wird über ein API Gateway angesprochen. Zur Authentifikation werden API-Keys verwendet, diese werden vom API Gateway generiert und geprüft. 

Für die Kontaktverwaltung wird eine PostgreSQL-Datenbank verwendet, als AWS-Produkt wird Amazon Relational Database Service (RDS) eingesetzt. 

Zur Speicherung der Spielberichte und des entsprechenden Spielverlaufs werden die Dateien in einem Amazon Simple Storage Service (S3) Bucket abgelegt. 

Die Matches werden zur Auffindbarkeit und späteren Wiederverwendbarkeit in einer MongoDB bei MongoDB Atlas gespeichert. Hier wurde sich gegen AWS entschieden, da die MongoDB direkt beim Anbieter kostenfrei ist, bei AWS würden sich die Kosten bei etwa 70 Euro monatlich einpegeln. 

## Funktionen
### Health Check
Zur Prüfung, ob die API zurzeit erreichbar ist, gibt es den HealthCheckController. Der Call kann so aussehen: 

```curl
curl --location 'https://dblv.saschahei.nl/api/hc'
```

Antwortet die API mit dem Status `200 OK`, ist die API erreichbar. 

### Kontaktverwaltung
Für die Kontaktverwaltung stehen diese Funktionen zur Verfügung

- POST New contact
- GET All contacts
- GET Contact by ID
- PATCH Contact by ID
- DELETE Contact by ID

Wie genau diese Funktionen zu verwenden sind, ist in der OpenAPI-Spezifikation definiert. Hierfür kann die `swagger.json` in diesem Repo verwendet werden. Ich empfehle, dafür [Redocly](https://redocly.github.io/redoc/?url=https://raw.githubusercontent.com/hnlvc/matchimporter-docs/main/swagger.json) zu verwenden. Grundsätzlich wird nachdem der API-Call abgesetzt wird ein Request an den Mediator gesandt, wodurch der UseCase ausgeführt wird. Hier ein Beispiel bei der Anlage von Kontakten: 

```csharp
    public async Task<ActionResult<TeamContactResponseContract>> CreateTeamContactAsync([FromBody] CreateTeamContactRequestContract request,
        CancellationToken cancellationToken)
    {
        var query = new CreateTeamContactCommand(
            request.TeamName,
            request.ClubName,
            request.Salutation,
            request.FirstName,
            request.LastName,
            request.MailAddress);
        var result = await _mediator.Send(query, cancellationToken);

        return Accepted(_mapper.Map<TeamContactResponseContract>(result));
    }
```

Zunächst wird ein `CreateTeamContactCommand` instanziiert, der die wichtigsten Informationen enthält. Das Ergebnis (`var result`) kommt dann vom Mediator. 

Im `CreateTeamContactUseCase` wird dann zunächst geprüft, ob der Kontakt bereits besteht und falls nicht, wird er angelegt. Das gleiche Verhalten zeigt sich beim Löschen oder Ändern von vorhandenen Kontakten. 

### Matchverwaltung
Für die Verwaltung von Matches stehen zurzeit zwei Endpunkte zur Verfügung:

- POST New Match
- GET Match by ID

Auch hier ist die Verwendung in der OpenAPI-Spezifikation (`swagger.json`) hinterlegt, die man mit [Redocly](https://redocly.github.io/redoc/?url=https://raw.githubusercontent.com/hnlvc/matchimporter-docs/main/swagger.json) anschauen kann. Ebenfalls wird hier je ein Request instanziiert, der an den Mediator gesandt, um den Use Case auszuführen.

Beim POST Call ist hier bewusst die Struktur etwas offener gestaltet, sodass an die API jegliche JSON-Dateien geschickt werden können. Die Entscheidung wurde getroffen, um mit den verschiedenen Formaten von BUP-Exporten besser umgehen zu können. Der Call an den Mediator sieht im Controller dann so aus:
```csharp
    public async Task<ActionResult> CreateMatchAsync([FromBody] CreateMatchRequestContract request, CancellationToken cancellationToken)
    {
        var command = new CreateMatchCommand(request.PdfFile, request.MatchJson);
        var match = await _mediator.Send(command, cancellationToken);
        
        return Accepted(_mapper.Map<MatchResponseContract>(match));
    }
```

Innerhalb des `CreateMatchUseCase` wird dann geprüft, was eine Art von BUP-Export vorliegt: 
```csharp
        if (request.MatchJson.RootElement.TryGetProperty("type", out var bupRequestType) &&
            !string.IsNullOrEmpty(bupRequestType.GetString()))
        {
            request.MatchJson.RootElement.TryGetProperty("event", out bupEventAsJson);
            bupEvent = bupEventAsJson.Deserialize<BupEvent>(_serializerOptions) ?? throw new InvalidOperationException();
        }
        else if (request.MatchJson.RootElement.TryGetProperty("date", out var matchDateAsJson) &&
                 !string.IsNullOrEmpty(matchDateAsJson.GetString()))
        {
            bupEvent = request.MatchJson.RootElement.Deserialize<BupEvent>(_serializerOptions);
        }
```

Sind die Properties, auf die geprüft wird, nicht vorhanden, geht die Applikation derzeit davon aus, dass es kein BUP-Export ist, ebenso wenn die Non-Nullable Properties der BupEvent-Klasse fehlen.
