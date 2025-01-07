<script>
  import { onMount } from "svelte"; // Importiert den Lifecycle-Hook `onMount` aus Svelte.
  import { songlist } from "./songlist.js"; // Importiert eine Liste von Songs aus einer separaten Datei.

  // Deklaration von Variablen zur Steuerung des Spiels.
  let songsrc = ""; // URL des aktuellen Songs (Vorschau oder vollständiger Track).
  let songprev = ""; // Vorschau-URL des Songs.
  let songid = ""; // ID des aktuellen Songs.
  let songname = ""; // Name des Songs.
  let points = 0; // Punktestand des Spielers.
  let query = ""; // Suchanfrage für die Songsuche.
  let searchResults = []; // Ergebnisse der Songsuche.
  let accessToken = ""; // Token zur Authentifizierung bei der Spotify-API.
  let audio; // Audio-Objekt zum Abspielen der Songs.
  let lives = 3; // Anzahl der Leben des Spielers.
  let nopoints = false; // Gibt an, ob Punkte für den aktuellen Song bereits vergeben wurden.

  // Spotify-API-Konstanten
  const CLIENT_ID = "2c857c676e2c4f869b35457808647994"; // Spotify-API-Client-ID.
  const CLIENT_SECRET = "91ffa986a186431494c123d629972cc8"; // Spotify-API-Client-Secret.
  const SPOTIFY_API_URL = "https://api.spotify.com/v1/search"; // Basis-URL für die Songsuche.

  // Initialisiert einen zufälligen Song aus der `songlist`.
  function RandomSong() {
    searchResults = []; // Löscht vorherige Suchergebnisse.
    query = ""; // Setzt die Suchanfrage zurück.
    nopoints = false; // Ermöglicht das Punkteverdienen für den neuen Song.
    const randomIndex = Math.floor(Math.random() * songlist.length); // Zufällige Auswahl eines Songs.
    songsrc = songlist[randomIndex].url;
    songid = songlist[randomIndex].id;
    songprev = songlist[randomIndex].previewUrl;
    songname = songlist[randomIndex].name;
    document.getElementById("songPlayer").style.visibility = "hidden"; // Versteckt den Player, bis der Song erraten wurde.
  }

  // Zeigt oder versteckt einen Lade-Spinner.
  function toggleSpinner(show) {
    const spinner = document.getElementById("animation");
    spinner.style.display = show ? "block" : "none"; // Anzeige des Spinners basierend auf der `show`-Variable.
  }

  // Holt ein Zugangstoken von der Spotify-API. Diese Funktion wurde mit SImon und mit ChatGPT erstellt
  async function fetchAccessToken() {
    try {
      const response = await fetch("https://accounts.spotify.com/api/token", {
        method: "POST",
        headers: {
          "Content-Type": "application/x-www-form-urlencoded",
          Authorization: `Basic ${btoa(CLIENT_ID + ":" + CLIENT_SECRET)}`, // Basis-Authentifizierung mit Client-ID und Secret.
        },
        body: "grant_type=client_credentials", // API-Anfrage für den Token-Typ "client_credentials".
      });

      if (!response.ok) {
        throw new Error("Access token retrieval failed"); // Fehlerbehandlung bei fehlgeschlagener Anfrage.
      }

      const data = await response.json();
      accessToken = data.access_token; // Speichert den erhaltenen Token.
    } catch (error) {
      console.error("Failed to fetch access token:", error);
    }
  }

  // Sucht Songs basierend auf der `query`-Variable.
  async function searchSongs() {
    if (!query) return; // Verhindert eine Suche ohne Eingabe.

    try {
      if (!accessToken) {
        await fetchAccessToken(); // Holt ein Token, falls noch keines vorhanden ist.
      }

      const response = await fetch(
        `${SPOTIFY_API_URL}?q=${encodeURIComponent(query)}&type=track`,
        {
          headers: {
            Authorization: `Bearer ${accessToken}`, // Authentifiziert die Anfrage mit dem Zugangstoken.
          },
        }
      );

      if (!response.ok) {
        throw new Error("Search failed"); // Fehlerbehandlung bei fehlgeschlagener Suche.
      }

      const data = await response.json();
      searchResults = data.tracks?.items || []; // Speichert die gefundenen Songs.
    } catch (error) {
      console.error("Error during search:", error);
    }
  }

  // Überprüft, ob die ID des geratenen Songs mit der des aktuellen Songs übereinstimmt.
  function checkMatch(id) {
    console.log("Guessed Song: " + id + " Real Song: " + songid);
    if (nopoints) {
      alert("Please move on to the next song."); // Verhindert weitere Aktionen, wenn Punkte bereits vergeben wurden.
      return;
    }

    if (id === songid) {
      nopoints = true; // Markiert den Song als bereits erraten.
      toggleSpinner(false); // Versteckt den Spinner.
      points++; // Erhöht die Punktzahl.
      document.getElementById("songPlayer").style.visibility = "visible"; // Zeigt den Player an.
      if (audio) audio.pause(); // Stoppt die Audiowiedergabe.
      alert("Correct! You earned a point."); // Informiert den Spieler über den Erfolg.
    } else {
      lives--; // Reduziert die Leben bei falscher Antwort.
      if (lives <= 0) {
        toggleSpinner(false);
        if (audio) audio.pause();
        alert(`Game Over! You ran out of lives! You scored ${points} points.`);
        restartGame(); // Startet das Spiel neu, wenn keine Leben mehr übrig sind.
      } else {
        alert(`Incorrect! You have ${lives} lives left.`); // Informiert den Spieler über verbleibende Leben.
      }
    }
  }

  // Spielt den Song ab.
  function playSong() {
    if (audio) {
      audio.pause(); // Stoppt vorherige Audiodateien.
    }
    audio = new Audio(songprev); // Erstellt ein neues Audio-Objekt für die Vorschau.
    audio
      .play()
      .then(() => {
        toggleSpinner(true); // Zeigt den Lade-Spinner an.
      })
      .catch((error) => {
        console.error("Audio playback failed:", error);
      });
  }

  // Stoppt die Audiowiedergabe.
  function stopSong() {
    if (audio) {
      audio.pause();
    }
    toggleSpinner(false); // Versteckt den Spinner.
  }

  // Überspringt zum nächsten Song.
  function nextSong() {
    RandomSong();
    if (audio) audio.pause();
    toggleSpinner(false);
  }

  // Startet das Spiel komplett neu.
  function restartGame() {
    points = 0;
    lives = 3;
    toggleSpinner(false);
    audio.pause();
    RandomSong();
  }

  // Initialisiert das Spiel beim Laden der Seite.
  onMount(() => {
    console.log(songlist.length);
    fetchAccessToken(); // Holt ein Zugangstoken.
    RandomSong(); // Wählt einen zufälligen Song.
  });
</script>

<main>
  <!-- Hauptüberschrift des Spiels -->
  <h1>MusixGuessr</h1>

  <!-- Unterüberschrift, die die Spielaufgabe beschreibt -->
  <h2>Guess the Song!</h2>

  <!-- Punkte- und Lebensanzeige:
       Punkte werden dynamisch aus der `points`-Variable angezeigt.
       Für die verbleibenden Leben werden Symbole (💙) verwendet,
       basierend auf der Anzahl von `lives`. -->
  <h3>
    Points: {points} | Lives: {#each Array(lives) as _}<span>💙</span>{/each}
  </h3>

  <!-- Spinner wird angezeigt, um zu visualisieren, dass etwas lädt (z. B. beim Abspielen eines Songs) -->
  <div id="animation" class="spinner"></div>

  <!-- Bereich für den Song-Player:
       Der `iframe` enthält den eingebetteten Player für den aktuellen Song,
       dessen Quelle (`src`) dynamisch aus der `songsrc`-Variable gesetzt wird.
       Der `style`-Attribut sorgt für eine abgerundete Darstellung. -->
  <div class="song">
    <iframe
      title="song"
      id="songPlayer"
      style="border-radius: 12px"
      src={songsrc}
      width="100%"
      height="152"
      frameBorder="0"
      allow="autoplay; clipboard-write; encrypted-media; fullscreen; picture-in-picture"
      loading="lazy"
    ></iframe>
  </div>

  <!-- Steuerungstasten für den Spieler:
       - "Next Song" lädt ein neues Lied.
       - "Play" spielt das Vorschau-Audio ab.
       - "Stop" stoppt die Audio-Wiedergabe. -->
  <button on:click={nextSong}>Next Song</button>
  <button on:click={playSong}>Play</button>
  <button on:click={stopSong}>Stop</button>

  <!-- Suchfeld:
       Benutzer können den Namen eines Songs eingeben, um danach zu suchen.
       Die Eingabe wird über `bind:value` an die `query`-Variable gebunden,
       und bei Änderungen wird `searchSongs` aufgerufen. -->
  <h2>Search for a Song</h2>
  <input
    type="text"
    placeholder="Search song..."
    bind:value={query}
    on:input={searchSongs}
  />

  <!-- Liste der Suchergebnisse:
       Wird dynamisch aus der `searchResults`-Liste generiert.
       Jedes Listenelement zeigt den Namen des Songs und den/die Künstler an.
       Durch Klicken auf einen Song wird `checkMatch` mit der Song-ID aufgerufen,
       um zu prüfen, ob die Wahl korrekt ist. -->
  <ul>
    {#each searchResults as song (song.id)}
      <li>
        <button on:click={() => checkMatch(song.id)}>
          {song.name} by {song.artists.map((artist) => artist.name).join(", ")}
        </button>
      </li>
    {/each}
  </ul>

  <!-- Schaltfläche zum Neustarten des Spiels:
       Setzt Punkte, Leben und den aktuellen Song zurück. -->
  <button id="restart-button" on:click={restartGame}>Restart</button>

  <!-- Ein dekoratives GIF in der unteren Ecke:
       Dient nur zur Verschönerung der Benutzeroberfläche. -->
  <img
    alt=""
    src="https://media.tenor.com/iwky5iDsEEkAAAAj/musical-notes.gif"
    class="corner-gif"
  />
</main>

<style>
  /*Im CSS Code wurden vereinzelte Code Teile von KI generiert übernommen.*/
  main {
    text-align: center;
    margin: auto; /* Zentriert den gesamten Container horizontal. */
    max-width: 500px;
    position: relative; /* Positionierungsbasis für absolute Kinder. */
    width: 800px; /* Breite des Containers. */
    height: 1200px; /* Höhe des Containers. */
    overflow: hidden; /* Verhindert, dass Inhalte über die Containergrenzen hinausragen. */
  }

  main::before {
    content: "";
    position: fixed; /* Fixiert das Pseudoelement relativ zum Ansichtsfenster. */
    top: 45%; /* Positioniert das Pseudoelement vertikal. */
    left: 63.5%; /* Positioniert das Pseudoelement horizontal. */
    transform: translate(
      -50%,
      -50%
    ); /* Verschiebt das Element so, dass es genau an der Position von `top` und `left` zentriert ist. */
    background-image: url("logo.png"); /* Setzt ein Hintergrundbild für das Pseudoelement. */
    background-repeat: no-repeat; /* Verhindert, dass das Hintergrundbild wiederholt wird. */
    background-size: contain; /* Passt das Hintergrundbild proportional in den Container ein. */
    width: 55vw; /* Breite des Pseudoelements als 55% der Ansichtsfensterbreite. */
    height: 55vh; /* Höhe des Pseudoelements als 55% der Ansichtsfensterhöhe. */
    opacity: 0.1; /* Macht das Pseudoelement halbtransparent. */
    z-index: -1; /* Lässt das Pseudoelement im Hintergrund erscheinen. */
  }

  h1 {
    font-size: 3.5rem; /* Schriftgrösse des Titels. */
    font-weight: bold; /* Setzt den Schriftstil auf fett. */
    background: linear-gradient(
      to right,
      lightgreen,
      darkblue
    ); /* Erstellt einen Farbverlauf für den Text. */
    -webkit-background-clip: text; /* Beschränkt den Farbverlauf auf den Text (Webkit-Browser). */
    background-clip: text; /* Beschränkt den Farbverlauf auf den Text (andere Browser). */
    -webkit-text-fill-color: transparent; /* Entfernt die Standard-Füllfarbe des Textes. */
  }

  .song {
    margin: 20px 0; /* Setzt vertikale Aussenabstände. */
    visibility: hidden; /* Lässt das Element unsichtbar, ohne den Layoutplatz freizugeben. */
  }

  button {
    padding: 10px 20px; /* Fügt innenliegende Abstände zu den Schaltflächen hinzu. */
    font-size: 16px; /* Schriftgrösse der Schaltflächen. */
    margin: 5px; /* Fügt Aussenabstände zwischen den Schaltflächen hinzu. */
  }

  .corner-gif {
    position: fixed; /* Fixiert das GIF an der angegebenen Position relativ zum Ansichtsfenster. */
    bottom: 10px; /* Abstand vom unteren Rand des Ansichtsfensters. */
    right: 10px; /* Abstand vom rechten Rand des Ansichtsfensters. */
    width: 50px; /* Breite des GIFs. */
  }

  #restart-button {
    position: fixed; /* Fixiert die Schaltfläche am unteren linken Rand. */
    bottom: 10px; /* Abstand vom unteren Rand des Ansichtsfensters. */
    left: 10px; /* Abstand vom linken Rand des Ansichtsfensters. */
  }

  input {
    padding: 10px; /* Innenabstand innerhalb des Eingabefelds. */
    font-size: 16px; /* Schriftgrösse im Eingabefeld. */
    width: 80%; /* Breite des Eingabefelds als Prozentsatz des Containers. */
    margin-bottom: 10px; /* Aussenabstand unterhalb des Eingabefelds. */
  }

  ul {
    list-style: none; /* Entfernt Aufzählungszeichen. */
    padding: 0; /* Entfernt die Standardauffüllung des UL-Elements. */
  }

  li {
    margin: 5px 0; /* Fügt vertikale Abstände zwischen den Listenelementen hinzu. */
  }

  .spinner {
    display: none; /* Standardmässig wird der Spinner ausgeblendet. */
    position: absolute; /* Positioniert den Spinner relativ zum übergeordneten Container. */
    top: 23%; /* Vertikale Position des Spinners im Container. */
    left: 43%; /* Horizontale Position des Spinners im Container. */
    width: 60px; /* Breite des Spinners. */
    height: 60px; /* Höhe des Spinners. */
    border: 6px solid rgba(0, 123, 255, 0.2); /* Definiert einen transparenten Rahmen für den Spinner. */
    border-top: 6px solid #007bff; /* Setzt die Farbe des oberen Randes für den Animationseffekt. */
    border-right: 6px solid #4caf50; /* Setzt die Farbe des rechten Randes für eine Multi-Farben-Drehung. */
    border-radius: 50%; /* Wandelt das quadratische Element in einen Kreis um. */
    animation: spin 1s cubic-bezier(0.68, -0.55, 0.27, 1.55) infinite; /* Startet eine flüssige Rotation mit einer benutzerdefinierten Beschleunigungskurve. */
  }

  @keyframes spin {
    0% {
      transform: rotate(0deg); /* Startet die Drehung bei 0 Grad. */
    }
    50% {
      transform: rotate(
        180deg
      ); /* Setzt die Drehung in der Hälfte des Zyklus auf 180 Grad. */
    }
    100% {
      transform: rotate(
        360deg
      ); /* Beendet die Drehung bei 360 Grad, was einer vollständigen Umdrehung entspricht. */
    }
  }
</style>
