# CineTrack - Vodič za odbranu Backend dela (API)

Ovaj dokument je osmišljen kao tvoj glavni podsetnik za odbranu projekta. Pokriva sve bitne aspekte backend aplikacije, podeljene po slojevima arhitekture.

Projekat koristi **N-tier (slojevitu) arhitekturu** u ASP.NET Core-u. Sistem je podeljen u četiri glavne celine:
1. **Modeli** (Preslikavaju strukturu baze)
2. **DTO objekti** (Komunikacija sa klijentom/frontend-om)
3. **Servisi** (Poslovna logika i rad sa bazom)
4. **Kontroleri** (HTTP API ulazne tačke)

---

## 1. Modeli (Models)

### Čemu služe i gde se koriste?
Modeli su C# klase koje reprezentuju strukturu tvojih podataka **u MongoDB bazi**. Oni definišu kako se podatak čuva. Koriste se prvenstveno u **Servisima**, gde servis uzme model, popuni ga podacima i upiše u bazu, ili ga pročita iz baze. Karakteriše ih upotreba `[BsonId]` atributa koji povezuje `Id` polje sa automatski generisanim MongoDB `ObjectId`-jem.

### Koji su modeli u projektu?
*   **`Movie`**: Predstavlja film. Sadrži atribute kao što su naslov, godina, žanrovi, režiser, URL postera. Takođe drži polja za prosečnu ocenu (`AverageRating`) i ukupan broj recenzija (`ReviewCount`).
*   **`User`**: Predstavlja korisnika. Sadrži username, email, heširanu lozinku (`PasswordHash`), URL avatara i `IsAdmin` bool polje koje određuje da li je korisnik administrator.
*   **`Review`**: Predstavlja recenziju koju je korisnik ostavio za film. Čuva ocenu (Rating), tekstualni opis (Content), kao i strane ključeve (`MovieId`, `UserId`). Pored toga, sadrži ugnježdenu listu komentara (`List<Comment> Comments`).
*   **`Comment`**: Pojedinačan komentar ispod recenzije.
*   **`WatchlistItem`**: Predstavlja "za gledanje" stavku, povezuje `UserId` sa `MovieId` i poseduje polje `Watched` (da li je odgledano) i vreme kada je dodato.

---

## 2. DTO klasni fajlovi (Data Transfer Objects)

### Čemu služe i zašto se koriste?
DTO fajlovi su samo "nosioci podataka". Klijent (Frontend) ne treba i ne sme da dobije ili pošalje pun Model iz baze (zbog bezbednosti i performansi). Na primer, kada korisnik kreira nalog, on šalje običnu lozinku, ali mi u bazu (Model) čuvamo heširanu. Takođe, ne želimo klijentu da šaljemo sistemska polja poput `CreatedAt`. Zato se podaci mapiraju iz DTO (koji primimo iz Frontenda) u Model (koji čuvamo u bazu).

### Koji su najbitniji DTO fajlovi?
*   **`RegisterUserDto` / `LoginUserDto`**: Šalju se sa frontenda prilikom kreiranja naloga ili prijave (sadrže Email, Password, Username).
*   **`AuthResponseDto`**: Vraća se klijentu usled uspešne prijave. Sadrži JWT (JSON Web Token) i osnovne informacije (Username, Email, IsAdmin) koje frontend smešta u lokalnu memoriju.
*   **`CreateMovieDto` / `UpdateMovieDto`**: Koristi ih Admin pri unosu filma. Sadrže naslov, režisera, godinu itd. (Samo ono što čovek unosi).
*   **`CreateReviewDto`**: Očekuje samo `MovieId`, `Rating` (1-5) i `Content`. ID ulogovanog korisnika kontroler vadi iz JWT tokena, ne iz DTO-a (radi bezbednosti, da ne bi neko lažirao recenzije za drugog).
*   **`TmdbImportRequestDto`**: Podaci koje Admin šalje kada pokreće import filmova sa javnog API-ja (stranice, jezik).

---

## 3. Servisi (Services)

### Čemu služe i gde se koriste?
Servisi sadrže čitavu **biznis logiku (business logic)** aplikacije. Njihov zadatak je da direktno manipulišu MongoDB kolekcijama. Koriste se (instanciraju se / Dependency Injection) unutar **Kontrolera**. Kontroler primi zahtev, a zatim "pita" servis da odradi stvarni posao i vrati mu rezultat.

### Koji su servisi u projektu i šta rade?
*   **`MovieService`**: Obavlja CRUD operacije za filmove. Najbitnije: poseduje metodu `GetAllAsync` koja radi pretragu (po nazivu i režiseru), filtriranje po žanru i omogućava **paginaciju** (Skip i Limit). Takođe sadrži metodu `UpdateRatingStatsAsync` koja preračunava prosek ocena na filmu.
*   **`UserService`**: Upravlja korisnicima. Proverava da li su username/email zauzeti, radi hesiranje lozinke koristeći **BCrypt** (jak algoritam za čuvanje šifri) i sadrži metodu `GenerateJwtToken` koja generiše kriptovani token za autorizaciju sa ugrađenim claim-ovima (ID, Uloga Admina).
*   **`ReviewService`**: Manipuliše recenzijama. Ono što je ovde jako bitno: *Svaki put kada se recenzija doda, promeni ili obriše*, ovaj servis poziva `MovieService.UpdateRatingStatsAsync` da bi u realnom vremenu apdejtovao prosečnu ocenu filma na koji se recenzija odnosi!
*   **`WatchlistService`**: Servis za čuvanje lične liste filmova koje korisnik planira da pogleda.
*   **`CommentsService`**: Menja komentare unutar konkretne recenzije. Komentari su ovde realizovani kao ugnježdeni dokumenti unutar `Review` kolekcije.
*   **`TmdbImportService`**: Ovo je "integracioni" servis. On preko ugrađenog `HttpClient`-a šalje HTTP zahteve spoljašnjem serveru (TMDb - The Movie Database). Preuzima listu popularnih filmova, parsira JSON odgovor i ubacuje ih u lokalnu bazu sprečavajući upisivanje duplikata.

---

## 4. Kontroleri (Controllers)

### Šta rade i čemu služe?
Kontroleri su klase zadužene za definisanje HTTP "ruta" (endpoint-ova), na pr. `/api/movies`. Njihova uloga je uska: primaju HTTP zahtev (`GET`, `POST`, `PUT`, `DELETE`), vrše osnovnu validaciju (često iz DTO objekata), obezbeđuju autorizaciju (ko sme da pristupi endpoint-u), prosleđuju parametre odgovarajućem **Servisu**, i na kraju vraćaju HTTP status kod (poput `200 OK`, `201 Created`, `400 BadRequest`, `404 Not Found`).

### Koji su kontroleri u projektu?
*   **`MoviesController`**:
    *   Svi korisnici mogu da gađaju `GET /api/movies` (lista, paginacija, pretraga) i da dobiju pojedinačni film i njegove recenzije.
    *   Moraš staviti anotaciju `[Authorize(Roles = "Admin")]` na `POST`, `PUT` i `DELETE` metode, jer samo administratori smeju da dodaju, menjaju ili brišu filmove iz sistema.
*   **`UsersController`**:
    *   `POST /api/users/register` – Kreiranje korisnika (proziva `UserService` da hesira lozinku i upiše u bazu).
    *   `POST /api/users/login` – Login endpoint, proziva verifikaciju lozinke i vraća klijentu generisan **JWT Token**.
    *   Sadrži rute za ažuriranje avatara (`/me/avatar`) i gledanje tuđih profila.
*   **`ReviewsController`**:
    *   Sadrži `[Authorize]` čime obezbeđuje da isključivo ulogovani korisnici mogu da ostave ocenu/recenziju na film.
    *   U okviru njega se iz JWT tokena preko `User.FindFirstValue(ClaimTypes.NameIdentifier)` vadi ID onoga ko šalje zahtev, što onemogućava lažiranje identiteta. Admin može da briše tuđe recenzije, obični useri samo svoje.
*   **`WatchlistController`**: Omogućava dodavanje filma u listu, brisanje i označavanje sa "Odgledao sam". Filter uvek gleda u JWT kako bi pokazao watchlistu isključivo onog korisnika koji je logovan.
*   **`CommentsController`**: Služi za rute tipa `/api/reviews/{id}/comments` – ostavljanje odgovora na nečiju recenziju.
*   **`ImportController`**: Sadrži rutu namenjenu isključivo Adminima za pokretanje onog `TmdbImportService`-a.

---
### Kratak "Flow" (Tok podataka) za usmeni primer:
Kada Admin želi da doda nov film:
1.  Admin na frontendu klikne "Dodaj" (šalje se JSON format pod nazivom `CreateMovieDto` na `POST /api/movies`).
2.  Zahtev pogađa **`MoviesController`** (koji ima `[Authorize(Roles="Admin")]` zaštitu).
3.  Kontroler prebacuje (mapira) podatke iz DTO-a u pravi Model (`Movie`).
4.  Kontroler poziva **`MovieService.CreateAsync(movie)`**.
5.  Servis setuje vreme kreiranja, prosečnu ocenu na 0 i preko MongoDB drajvera to trajno sačuva u bazi.
6.  Servis vraća sačuvan objekat, kontroler vraća status kod `201 Created` ka frontendu.
