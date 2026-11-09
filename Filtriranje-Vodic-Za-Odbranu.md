# Filtriranje u MongoDB-u (CineTrack Projekat)

Ovaj dokument sadrži sva objašnjenja vezana za ulogu i implementaciju filtriranja (pretraga i žanrovi) u CineTrack API-ju.

---

## 1. Šta je filtriranje i koja mu je uloga?
Filtriranje služi da iz kolekcije (gde možda ima hiljade ili milione dokumenata) izvučeš **samo one dokumente koji ispunjavaju određene uslove**, umesto da bazu pitaš za sve podatke odjednom. 

Njegova glavna uloga je **optimizacija performansi i resursa**:
* **Ušteda memorije (RAM-a):** Ako imaš 10,000 filmova, a korisnik traži samo akcione, filtriranjem u bazi vraćaš samo npr. 50 filmova u radnu memoriju (RAM) tvoje C# aplikacije.
* **Brzina (Mreža):** Manje podataka se prenosi kroz mrežu između MongoDB servera i tvog Backend API-ja, što drastično ubrzava vreme odziva.

## 2. Zašto filtriramo u bazi (MongoDB-u), a ne u C#-u?
Ponekad studenti naprave grešku pa kažu bazi "daj mi sve filmove" (`_movies.Find(_ => true).ToList()`), a onda u C#-u koriste LINQ (npr. `.Where(m => m.Genre == "Action")`) da ih profiltriraju. To je **antipattern** (loša praksa). 

MongoDB je specijalizovan za pretragu i ima indekse koji mu omogućavaju da milisekundama pronađe tražene podatke. Tvoj C# API bi „pukao” pod opterećenjem kada bi morao da učita stotine hiljada dokumenata u memoriju samo da bi odbacio 99% njih. Zato bazi prosleđujemo "filter" (upit) pre nego što nam vrati podatke.

---

## 3. Gde je tačno implementirano filtriranje?

Filtriranje po *search baru* i po *žanrovima* implementirano je na dva mesta: krene od **Kontrolera** (gde se podaci primaju) i završi se u **Servisu** (gde se pravi stvarni upit za bazu).

### Korak A: Prijem parametara u Kontroleru
**Fajl:** `MovieTracker.API/Controllers/MoviesController.cs`
**Metoda:** `GetAll` (HTTP GET na `/api/movies`)

Kada front-end (React aplikacija) pošalje zahtev sa parametrima za pretragu (npr. `/api/movies?search=batman&genre=Action`), kontroler ih prihvata preko `[FromQuery]` atributa:

```csharp
[HttpGet]
public async Task<IActionResult> GetAll([FromQuery] string? search, [FromQuery] string? genre, [FromQuery] int page = 1, [FromQuery] int pageSize = 10)
{
    // ... validacija paginacije ...

    // Ovde kontroler prosleđuje 'search' i 'genre' u servis:
    var (items, total) = await _movieService.GetAllAsync(search, genre, page, pageSize);
    // ...
}
```

### Korak B: Glavni protok u GetAllAsync (Servis)
**Fajl:** `MovieTracker.API/Services/MovieService.cs`
**Metoda:** `GetAllAsync`

`GetAllAsync` dobija reč za pretragu i žanr iz kontrolera i obavlja proces u tri koraka:

```csharp
public async Task<(List<Movie> Items, long Total)> GetAllAsync(string? search, string? genre, int page, int pageSize)
{
    // 1. POZIVA POMOĆNU METODU DA NAPRAVI FILTER
    var filter = BuildFilter(search, genre);

    // 2. NALAZI KOLIKO UKUPNO FILMOVA ZADOVOLJAVA FILTER (za računanje broja stranica)
    var total = await _movies.CountDocumentsAsync(filter);
    
    // 3. NALAZI I VRAĆA SAME FILMOVE (sa primenjenim filterom i paginacijom)
    var items = await _movies.Find(filter)
        .SortByDescending(x => x.CreatedAt) // Sortira po datumu dodavanja
        .Skip((page - 1) * pageSize)        // Preskače prethodne stranice (Paginacija)
        .Limit(pageSize)                    // Uzima samo npr. 10 filmova za trenutnu stranicu (Paginacija)
        .ToListAsync();

    return (items, total);
}
```

### Korak C: Logika sklapanja filtera (BuildFilter)
**Fajl:** `MovieTracker.API/Services/MovieService.cs`
**Metoda:** `BuildFilter`

Ovo je mesto gde se tačno pravi upit (query) koji ide ka bazi:

```csharp
private static FilterDefinition<Movie> BuildFilter(string? search, string? genre)
{
    var filterBuilder = Builders<Movie>.Filter;
    var filter = filterBuilder.Empty; // Početni filter - "daj mi sve"

    // 1. Filtriranje po Search Baru
    if (!string.IsNullOrWhiteSpace(search))
    {
        var normalized = search.Trim();
        // Pravi case-insensitive Regex (nebitna mala/velika slova) za Naslov
        var titleFilter = filterBuilder.Regex(x => x.Title, new MongoDB.Bson.BsonRegularExpression(normalized, "i"));
        // Pravi case-insensitive Regex za Režisera
        var directorFilter = filterBuilder.Regex(x => x.Director, new MongoDB.Bson.BsonRegularExpression(normalized, "i"));
        
        // Spaja ih sa "ILI" (OR) - vrati filmove gde se tekst poklapa sa naslovom ILI režiserom
        filter &= filterBuilder.Or(titleFilter, directorFilter);
    }

    // 2. Filtriranje po Žanrovima
    if (!string.IsNullOrWhiteSpace(genre))
    {
        // Traži filmove gde niz 'Genres' u sebi sadrži traženi žanr
        filter &= filterBuilder.AnyEq(x => x.Genres, genre.Trim());
    }

    return filter;
}
```

---

## 4. Kako radi filtriranje po žanrovima ispod haube?

Tvoj model `Movie` u C#-u (i u bazi) ima svojstvo `Genres` koje je **lista** (niz stringova):
```csharp
public List<string> Genres { get; set; } = new();
```
Na primer, jedan film u bazi izgleda ovako: `Genres: ["Action", "Sci-Fi", "Drama"]`.

Kada korisnik sa frontenda traži žanr "Action", u kod stiže `genre = "Action"`.
Metoda `AnyEq(x => x.Genres, genre.Trim())` je specijalna komanda MongoDB drajvera koja bazi kaže: 
**"Prođi kroz niz `Genres` u svakom filmu i vrati mi ovaj dokument SAMO ako se BILO KOJI (`Any`) element iz tog niza POKLAPA (`Eq` - Equals) sa rečju 'Action'."**

Na kraju imamo onaj znak `&=` (kao logičko AND). 
To znači da ako je korisnik uneo i reč u search bar (npr. "Batman") i izabrao žanr (npr. "Action"), bazi će biti poslat jedan spojen upit: *"Vrati mi film koji u imenu ili režiseru sadrži 'Batman' **I** koji u nizu žanrova ima 'Action'."* Sve se to spakuje u onaj jedan `filter` koji se prosledi bazi.
