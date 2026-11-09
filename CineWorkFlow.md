# Tokovi rada (Workflows) u CineTrack Aplikaciji

U ovom dokumentu je detaljno, korak po korak, objašnjeno šta se dešava u pozadini (na Backend-u) od trenutka kada korisnik klikne na neku akciju na Frontend-u (klijentu), do trenutka kada mu Backend pošalje odgovor. Ovi tokovi su ključni za razumevanje kako sistem diše i kako komponente razgovaraju međusobno.

---

## 1. Autentifikacija: Registracija i Prijava (Login)

### A) Registracija novog korisnika
1. **Frontend:** Korisnik popunjava formu na sajtu i klika "Registruj se". Šalje se `POST` HTTP zahtev na `/api/users/register` sa telom (JSON) u kome su Email, Password i Username (`RegisterUserDto`).
2. **Kontroler (`UsersController`):** Prima taj zahtev. Odmah komunicira sa `UserService`-om i traži da se proveri da li je Email ili Username već prisutan u bazi. Ako jeste, odbija zahtev i vraća `400 BadRequest`.
3. **Servis (`UserService`):** Pravi se prazan C# objekat tipa `User`. Aplikacija uzima šifru u običnom tekstu (plaintext) i **NIKADA je ne čuva takvu u bazi**. Umesto toga poziva biblioteku `BCrypt` koja je ireverzibilno kriptuje i pretvara u `PasswordHash`. Korisnik se inicijalno označava kao običan (`IsAdmin = false`).
4. **Baza (`MongoDB`):** Preko MongoDB C# Drajvera `_users.InsertOneAsync`, novi korisnik se trajno upisuje u kolekciju.
5. **Odgovor:** Kontroler vraća klijentu status `200 OK` zajedno sa osnovnim podacima (ali BEZ lozinke!).

### B) Prijava (Login)
1. **Frontend:** Korisnik kuca podatke i šalje `POST /api/users/login` (`LoginUserDto`).
2. **Kontroler:** Zove servisnu metodu `ValidateCredentialsAsync`.
3. **Servis:** Traži u MongoDB-u korisnika sa tim Email-om. Kada ga nađe, poziva specijalnu metodu `BCrypt.Verify(password, passwordHash)` koja upoređuje unetu lozinku sa šifrovanim nizom karaktera u bazi.
4. **Generisanje Tokena (Ključna stvar):** Ako se lozinke poklapaju, poziva se `GenerateJwtToken(user)`. Backend kreira digitalnu "propusnicu" - **JWT (JSON Web Token)**. U nju spakuje ID korisnika, njegov Email, Username i informaciju o tome da li je **Admin** (ovo se nazivaju `Claims`). Zatim taj token kriptografski potpisuje tajnim ključem.
5. **Odgovor:** Backend vraća token frontendu. Frontend će od tog trenutka uz **svaki sledeći** zahtev slati ovaj token kako bi dokazao ko je (unutar `Authorization: Bearer <token>` headera).

---

## 2. Pregled filmova (Filtriranje i Paginacija)

1. **Frontend:** Otvara se lista filmova. Šalje se `GET /api/movies?search=Betmen&genre=Action&page=1&pageSize=10`.
2. **Kontroler (`MoviesController`):** Prima zahtev. Ova ruta nema `[Authorize]` atribut na sebi, pa svako može da je pogodi.
3. **Servis (`MovieService`):**
    * Poziva se `BuildFilter` koji od primljenih stringova sklapa logički `MongoDB Filter`. Koristi **Regularne Izraze (Regex)** kako ne bi bilo bitno da li korisnik kuca velika ili mala slova.
    * Zatim se okine `_movies.CountDocumentsAsync(filter)`. Baza prebroji koliko ukupno filmova ispunjava uslove. Frontend koristi ovaj broj da iscrta tačan broj "stranica" (dugmići 1, 2, 3...).
    * Okida se glavno dohvatanje podataka: `Find(filter)`. Primenjuje se Sortiranje (od najnovijih), zatim `.Skip()` (preskače filmove sa prethodnih strana) i `.Limit(10)` (uzima samo 10 komada). Ovo se zove **Paginacija baze podataka**. 
    * *Zašto paginacija?* Da ti aplikacija ne bi prsla ako imaš 20.000 filmova.
4. **Odgovor:** Kontroler pakuje `items` (filmove) i `total` u jedan anonimni objekat i vraća ga klijentu.

---

## 3. Sistem Recenzija i Prosečna Ocena (Reviews & Average Rating)
*(Ovo je fantastičan arhitektonski potez koji profesor obožava da čuje, fokusiraj se na ovo)*

1. **Frontend:** Ulogovan korisnik ostavlja recenziju na film. Šalje `POST /api/reviews` sa podacima (`MovieId`, `Rating`, `Content`).
2. **Kontroler (`ReviewsController`):** Uključuje se `[Authorize]` zaštita. Kontroler **NE VAMUJE** iz `Dto` objekta ko je ostavio recenziju. Umesto toga, iz bezbednosnih razloga, Backend čita onaj `JWT Token` koji je stigao, dešifruje ga i iz njega pouzdano vadi `UserId` (sprečavajući "hakere" da ostave recenziju u tuđe ime).
3. **Servis (`ReviewService`):**
    * Dodaje recenziju u `reviews` kolekciju.
    * **MAGIJA (Denormalizacija za performanse):** Umesto da prilikom svakog učitavanja početne stranice sistem "znoji" bazu i iznova izračunava proseke ocena spajajući `Movies` i `Reviews` tabele – on to radi SAMO KAD SE MENJA OCENA.
    * Odmah nakon što se recenzija sačuva, `ReviewService` poziva funkciju `RecalculateAverageRatingAsync`.
    * Ta funkcija pokupi sve ocene ikada ostavljene za taj film, preračuna prosek i ukupan broj, a onda kaže `MovieService`-u: "Ej, ažuriraj polje `AverageRating` direktno unutar samog filma".
    * Na ovaj način, kad neko samo pregleda filmove na Homepage-u, prosek ocena se pročita iz samog filma trenutno i ekstremno brzo.
4. **Odgovor:** Vraća status `201 Created` zajedno sa recenzijom.

---

## 4. Komentari ispod recenzija (Comments)

1. **Frontend:** Korisnik klikne "Odgovori" ispod nečije recenzije i pošalje tekst.
2. **Kontroler (`CommentsController`):** ID korisnika i Username se opet sigurno dobijaju iz JWT Tokena.
3. **Servis (`CommentsService`):**
    * Opet vrlo pametan pristup bazi. Umesto da se pravi nova kolekcija `Comments` i veže za `Reviews` pomoću stranog ključa (kao u SQL-u), koristi se specifičnost MongoDB-a (NoSQL) a to je **Ugnježdivanje (Embedding)**.
    * Servis kreira objekat komentara u C#-u i koristi brzu komandu: `Builders<Review>.Update.Push(x => x.Comments, comment)`.
    * Ta komanda kaže bazi: "Idi u konkretnu Recenziju, nađi njen niz `Comments` i samo uguraj ovaj novi komentar u njega". Nema učitavanja čitavog dokumenta u C# samo zbog izmene.

---

## 5. Lista za gledanje (Watchlist)

1. **Frontend:** Korisnik naiđe na film koji želi da pogleda kasnije i klikne srce/dugme. Šalje `POST /api/watchlist`.
2. **Servis (`WatchlistService`):**
    * Odlazi u bazu i proverava da li dokument u `watchlists` kolekciji sa tim `UserId` i `MovieId` već postoji. Ako postoji, baca grešku (nema dupliranja).
    * Preko `MovieService` dohvata film. Zašto? Zato što će u `WatchlistItem` dokument da **iskopira** naslov filma, sliku i godinu.
    * Kopiranje podataka u NoSQL bazi (Denormalizacija) se radi kako bi ruta `GET /api/watchlist` radila neverovatno brzo. Kad korisnik otvori svoju listu, baza mu odmah vraća sve potrebne podatke za prikaz liste, umesto da baza mora da juri `MovieId` iz watchliste po `Movies` kolekciji da bi našla sliku.
3. **Markiranje (Watched):** Kad korisnik klikne da je odgledao (`PATCH /api/watchlist/{id}/watched`), servis radi ciljani apdejt, menja polje `Watched` na `true` i snima vreme gledanja `WatchedAt = DateTime.UtcNow`.

---

## 6. Uvoz Filmova (TMDb Import)

1. **Frontend (Admin Panel):** Admin upisuje koliko stranica iz svetske baze The Movie Database (TMDb) želi da prekopira i pokreće akciju.
2. **Kontroler (`ImportController`):** Ima `[Authorize(Roles = "Admin")]`. Znači, ako običan korisnik i ukrade URL do ove rute, server mu vraća `403 Forbidden` zato što njegov JWT token ne sadrži privilegiju `Admin`.
3. **Servis (`TmdbImportService`):**
    * Ovo je tipičan "Integracioni" tok rada, gde tvoj Backend postaje Klijent nekom tuđem Backend-u.
    * Servis prvo dovuče imena svih filmova iz tvoje lokalne baze i stavi ih u brzi rečnik (`HashSet`), kako bi pri uvozu momentalno detektovao duplikate.
    * Koristeći sistemski `HttpClient`, tvoj kod šalje zahtev putem Interneta ka `api.themoviedb.org`.
    * TMDb vraća rezultat u JSON formatu. Tvoj C# kod ga **De-serijalizuje** (pretvara taj JSON tekstualni format nazad u pametne C# objekte).
    * Ako film zadovolji sve logičke provere (nema prazan naslov, ima validan datum i nije duplikat), dodaje se u `moviesToInsert` listu.
    * Na samom kraju, umesto da u petlji 100 puta gađa tvoju bazu (`InsertOne`), C# okida `InsertManyAsync(moviesToInsert)`. Ovo se zove **Bulk insert** - u jednom jedinom mrežnom putovanju ka bazi upisuje se stotine filmova. To neverovatno optimizuje aplikaciju.
