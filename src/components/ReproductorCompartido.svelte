<script>
    import { onMount } from 'svelte';
    import { writable, get } from 'svelte/store';
    import * as d3 from 'd3';
    
    // cuando se "reproduce" una canción -> juego que canción le gusta más al usuario.
    import { reproduccionesPorPersona } from "../stores.js"; 

    let allSongsData = [];
    const colorGenero = d3.scaleOrdinal()
        .domain(["Pop", "Rock", "Indie", "Electrónica", "Reggaetón", "Rap", "Balada", "Regional", "Trap", "Funk", "Cumbia Pop", "Rock Latino", "Metal"])
        .range(["#00CC66", "#CC0000", "#CC0066", "#001BCC", "#CCB400", "#000000", "#800080", "#FFA500", "#DAA520", "#FF1493", "#32CD32", "#6A5ACD", "#8B0000"]);

    const simboloSelector = {
        Var: "/images/Var.png",
        Rosita: "/images/Rosita.png",
        Steffy: "/images/Steffy.png",
        Cruz: "/images/Cruz.png",
        Igual: "/images/Igual.png"
    };

    export const userReproducers = writable(JSON.parse(localStorage.getItem('userReproducers') || '[]'));

    userReproducers.subscribe(value => {
        localStorage.setItem('userReproducers', JSON.stringify(value));
    });

    let userName = "";
    let songName = "";
    let artistName = "";
    let selectedGenre = "";
    let selectedColor = "#00CC66";
    let danzabilityInput = 0;
    let moodInput = "";

    // Variables que se actualizarán justo antes de crear el reproductor
    let assignedSymbol = ""; 
    let assignedSymbolText = "";
    let assignedSymbolDescription = "";
    let isCalculatingSymbol = true; 
    let hasPlayed = false; // Se gestionará en base a assignSymbolBasedOnAfinity

    let songSuggestions = [];
    let artistSuggestions = [];
    let genreSuggestions = [];

    onMount(async () => {
        try {
            allSongsData = await d3.csv("/cancionescompartidas.csv");
            const uniqueGenresFromCsv = [...new Set(allSongsData.map(d => d.Genero).filter(Boolean))];
            colorGenero.domain(Array.from(new Set([...colorGenero.domain(), ...uniqueGenresFromCsv])));
            // Calculamos el símbolo inicial al cargar, si ya hay datos de afinidad
            await calcularSimboloConDelay();
        } catch (error) {
            console.error("Error al cargar cancionescompartidas.csv:", error);
            isCalculatingSymbol = false;
            // Asegurarse de que si falla la carga, el símbolo sea la cruz (o sea no jugó la persona)
            assignedSymbol = simboloSelector.Cruz;
            assignedSymbolText = "Cruz";
            assignedSymbolDescription = " No jugaste todavía, por eso te identificás con una cruz.";
            hasPlayed = false;
        }
    });

    async function calcularSimboloConDelay() {
        isCalculatingSymbol = true;
        // Agregamos un pequeño delay para que el mensaje "Calculando..." del símbolo sea visible hasta que aparezca
        await new Promise(resolve => setTimeout(resolve, 2000)); 
        assignSymbolBasedOnAfinity(); // Calcula el símbolo basado en el estado actual
        isCalculatingSymbol = false;
    }

    function filterSuggestions(query, type) {
        if (!query) return [];
        const lowerQuery = query.toLowerCase();

        if (type === 'song') {
            return allSongsData.filter(d => d.Cancion && d.Cancion.toLowerCase().includes(lowerQuery))
                .map(d => d.Cancion).slice(0, 5);
        } else if (type === 'artist') {
            return [...new Set(allSongsData.filter(d => d.Artista && d.Artista.toLowerCase().includes(lowerQuery))
                .map(d => d.Artista))].slice(0, 5);
        } else if (type === 'genre') {
            return [...new Set(allSongsData.map(d => d.Genero))]
                .filter(g => g && g.toLowerCase().includes(lowerQuery)).slice(0, 5);
        }
        return [];
    }

    function handleSongInput(event) {
        songName = event.target.value;
        songSuggestions = filterSuggestions(songName, 'song');
        artistName = ""; // Limpiar si la canción cambia
        selectedGenre = ""; // Limpiar si la canción cambia
        danzabilityInput = 0; 
        moodInput = "";
    }

    // Idea Pre-listado:
    function selectSong(suggestion) {
        songName = suggestion;
        const foundSong = allSongsData.find(d => d.Cancion === suggestion);
        if (foundSong) {
            artistName = foundSong.Artista;
            selectedGenre = foundSong.Genero;
            selectedColor = colorGenero(selectedGenre);
            danzabilityInput = +foundSong.Danzabilidad;
            moodInput = foundSong.Mood;
        }
        songSuggestions = [];
    }

    function handleArtistInput(event) {
        artistName = event.target.value;
        artistSuggestions = filterSuggestions(artistName, 'artist');
    }

    function selectArtist(suggestion) {
        artistName = suggestion;
        artistSuggestions = [];
    }

    function handleGenreInput(event) {
        selectedGenre = event.target.value;
        selectedColor = colorGenero(selectedGenre);
        genreSuggestions = filterSuggestions(selectedGenre, 'genre');
    }

    function selectGenre(suggestion) {
        selectedGenre = suggestion;
        selectedColor = colorGenero(selectedGenre);
        genreSuggestions = [];
    }

    // Lógica para obtener los símbolos de acuerdo a nuestras afinidades
    function calcularSimbolo() {
		const current = get(reproduccionesPorPersona);
		const total = current.Steffy + current.Rosita + current.Var;

		if (total === 0) return asignarCruz();

		const afinidades = [
			{ nombre: "Steffy", porcentaje: current.Steffy / total, simbolo: simboloSelector.Steffy },
			{ nombre: "Rosita", porcentaje: current.Rosita / total, simbolo: simboloSelector.Rosita },
			{ nombre: "Var", porcentaje: current.Var / total, simbolo: simboloSelector.Var }
		];

		afinidades.sort((a, b) => b.porcentaje - a.porcentaje);
		const [a, b, c] = afinidades;

		if (a.porcentaje === b.porcentaje && b.porcentaje === c.porcentaje) {
			asignarIgual("Empate perfecto entre los tres.");
		} else if (a.porcentaje === b.porcentaje) {
			asignarIgual(`Empate entre ${a.nombre} y ${b.nombre}`);
		} else {
			assignedSymbol = a.simbolo;
			assignedSymbolText = a.nombre;
			assignedSymbolDescription = `Tu afinidad es con ${a.nombre}`;
		}
	}

	function asignarCruz() {
		assignedSymbol = simboloSelector.Cruz;
		assignedSymbolText = "Cruz";
		assignedSymbolDescription = "No jugaste todavía. ¡Jugá para conocer tu símbolo!";
	}

	function asignarIgual(texto) {
		assignedSymbol = simboloSelector.Igual;
		assignedSymbolText = "Empate";
		assignedSymbolDescription = texto;
	}

    
    function createReproducer() {
        if (!userName || !songName || !artistName || !selectedGenre || danzabilityInput === null || moodInput === "") {
            alert("Por favor, completá todos los campos.");
            return;
        }

        const finalDanzability = Math.max(0, Math.min(100, parseInt(danzabilityInput) || 0));
		calcularSimbolo();
        // La persona puede armar el reproductor personalizado:
		const nuevo = {
			id: Date.now(),
			userName,
			songName,
			artistName,
			genre: selectedGenre,
			color: selectedColor,
			danzability: finalDanzability,
			mood: moodInput,
			symbol: assignedSymbol,
			symbolText: assignedSymbolText
		};

		userReproducers.update(actual => [...actual, nuevo]);
        // Resetear el formulario
        userName = "";
        songName = "";
        artistName = "";
        selectedGenre = "";
        selectedColor = colorGenero("Pop"); // Volver a un color por defecto
        danzabilityInput = 0;
        moodInput = "";
        
    }

    function clearReproducers() {
        if (confirm("¿Borrar todos los reproductores?")) {
            userReproducers.set([]);
            // También deberías resetear reproduccionesPorPersona si borras todo
            reproduccionesPorPersona.set({ Steffy: 0, Rosita: 0, Var: 0 });
            // Y recalcular el símbolo para que muestre la cruz de nuevo
            assignSymbolBasedOnAfinity(); 
        }
    }

    function obtenerDiametroReproductorUsuario() {
        return 120;
    }

    function wavePathReproductorUsuario(danzability) {
        const danceability = parseFloat(danzability || 50);
        const amplitude = 1 + 5 * (danceability / 100);
        const frequency = 0 + 0.65 * (danceability / 100);
        const width = 100;
        const baseY = width * 0.16;

        let d = `M 0 ${baseY}`;
        for (let x = 0; x < width; x++) {
            const waveY = Math.sin(x * frequency);
            const y = baseY + waveY * amplitude;
            d += ` L ${x} ${y}`;
        }
        return d;
    }

    function strokeWidthReproductorUsuario() {
        return 2;
    }

    function getMoodEmoji(mood) {
        switch (mood.toLowerCase()) {
            case 'flow': return '🌊';
            case 'perreo': return '🔥';
            case 'trap': return '🚧';
            case 'fiesta': return '🎉';
            case 'club': return '🎶';
            case 'sexy': return '💋';
            case 'electrónico': return '⚡';
            case 'melancólico': return '😔';
            case 'divertido': return '😂';
            case 'bailable': return '💃';
            case 'alternativo': return '🤘';
            case 'retro': return '✨';
            case 'funky': return '🕺';
            case 'alegre': return '😊';
            case 'relajado': return '😌';
            case 'chill': return '🛋️';
            case 'romántico': return '❤️';
            case 'intenso': return '💥';
            case 'calle': return '🛣️';
            case 'empowerment': return '💪';
            case 'melódico': return '🎤';
            case 'oscuro': return '🖤';
            case 'épico': return ' epic';
            case 'legendario': return '👑';
            case 'veraniego': return '☀️';
            case 'glam': return '💎';
            case 'reflexivo': return '🤔';
            case 'powerful': return '⚡';
            case 'seductor': return '😏';
            case 'cool': return '😎';
            case 'historia': return '📜';
            case 'disco': return '🪩';
            case 'metal': return '🤘';
            default: return '🎧';
        }
    }


</script>

<style>
    /* Estilos generales para el componente */
    .reproductor-compartido-container {
        max-width: 900px;
        margin: 40px auto;
        margin-top: -95px;
        padding: 30px;
        padding-top: 30px;
        background-color: #f9f9f9;
        border-radius: 15px;
        box-shadow: 0 8px 20px rgba(0, 0, 0, 0.1);
        font-family: 'Raleway', sans-serif;
        color: #333;
        text-align: center;
    }

    .reproductor-compartido-container h2 {
        color: #003058;
        margin-bottom: 25px;
        font-size: 2.2em;
    }

    .form-crear-reproductor {
        display: grid;
        grid-template-columns: 1fr 1fr;
        gap: 20px;
        margin-bottom: 40px;
        text-align: left;
    }

    .form-group {
        position: relative;
    }

    .form-group label {
        display: block;
        margin-bottom: 8px;
        font-weight: bold;
        color: #555;
    }

    .form-group input {
        width: calc(100% - 20px); /* Ajuste para padding */
        padding: 12px 10px;
        border: 1px solid #ddd;
        border-radius: 8px;
        font-size: 1em;
        box-sizing: border-box; /* Incluye padding y borde en el ancho */
        transition: border-color 0.3s ease;
    }


    .form-group input:focus {
        border-color: #003058;
        outline: none;
    }

    .suggestions-list {
        position: absolute;
        top: 100%;
        left: 0;
        right: 0;
        background-color: white;
        border: 1px solid #ddd;
        border-top: none;
        border-radius: 0 0 8px 8px;
        z-index: 10; /* Asegura que las sugerencias estén por encima de otros elementos */
        max-height: 150px;
        overflow-y: auto;
        box-shadow: 0 4px 10px rgba(0, 0, 0, 0.08);
    }

    .suggestion-item {
        padding: 10px;
        cursor: pointer;
        text-align: left;
        transition: background-color 0.2s ease;
    }

    .suggestion-item:hover {
        background-color: #eee;
    }

    .color-input-wrapper {
        display: flex;
        align-items: center;
        gap: 10px;
        margin-top: 15px; /* Más espacio para separar del input de Género */
        padding-top: 10px; /* Padding para que el separador sea visible */
        border-top: 1px dashed #eee; 
    }

    .color-input-wrapper label {
        margin-bottom: 0; 
    }

    .color-input-wrapper input[type="color"] {
        width: 50px;
        height: 40px;
        padding: 0;
        border: none;
        cursor: pointer;
        border-radius: 5px;
        overflow: hidden; 
        margin-left: 10px; 
    }
    /* Estilos específicos para el color picker en navegadores WebKit (Chrome, Edge, Safari) */
    .color-input-wrapper input[type="color"]::-webkit-color-swatch-wrapper {
        padding: 0;
    }
    .color-input-wrapper input[type="color"]::-webkit-color-swatch {
        border: none;
    }

    /* Contenedor para el símbolo literal y el texto en el formulario */
    .symbol-display-container {
        display: flex;
        align-items: center;
        font-size: 1.2em;
        margin-top: 10px;
        color: #003058;
        margin-left:0px;
        margin-right:135px;
        gap: 6px;
        text-align: left;
        flex-wrap: wrap;
    }
    .assigned-symbol-preview {
        width: 40px; 
        height: 40px;
        object-fit: contain;
        filter: brightness(0); 
        margin-right:5px; 
        vertical-align: middle;
        text-align: left;
        margin-left: 10px;
        flex-shrink: 0; 
    }

    .symbol-display-container span.bold-symbol-text {
        font-weight: bold;
        margin-right: 5px;
    }

    .description-symbol {
        font-size: 0.85em;
        color: #777;
        margin-top: 5px;
        line-height: 1.4;
        grid-column: 1 / -1; 
        text-align: left; /* Alineado con el resto del formulario */
    }

    .btn-crear-reproductor {
        grid-column: 1 / -1; /* Ocupa ambas columnas */
        background-color: #15FB88;
        color: #000;
        padding: 14px 30px;
        border: none;
        border-radius: 25px;
        font-weight: bold;
        cursor: pointer;
        font-size: 1.1em;
        transition: background-color 0.3s ease, transform 0.2s ease, box-shadow 0.3s ease;
        margin-top: 20px;
        box-shadow: 0 4px 8px rgba(21, 251, 136, 0.4);
    }

    .btn-crear-reproductor:hover {
        background-color: #00CC66;
        transform: translateY(-2px);
        box-shadow: 0 6px 12px rgba(0, 204, 102, 0.5);
    }

    .btn-clear-reproducers {
        background-color: transparent;
        color: #cc0000;
        padding: 10px 20px;
        border: 2px solid #cc0000;
        border-radius: 8px;
        cursor: pointer;
        margin-top: 30px;
        font-size: 0.95em;
        font-weight: bold;
        transition: all 0.3s ease;
        display: block;
        margin-left: auto;
        margin-right: auto;
        letter-spacing: 0.5px;
        box-shadow: 0 2px 5px rgba(204, 0, 0, 0.2);
    }

    .btn-clear-reproducers:hover {
        background-color: #cc0000;
        color: white;
        box-shadow: 0 4px 10px rgba(204, 0, 0, 0.4);
        transform: translateY(-1px);
    }

    /* Estilos para la visualización de los reproductores guardados */
    .reproductores-guardados {
        margin-top: 50px;
        border-top: 1px solid #eee;
        padding-top: 30px;
    }

    .reproductores-guardados h3 {
        color: #003058;
        margin-bottom: 30px;
        font-size: 1.8em;
    }

    .lista-reproductores {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
        gap: 25px;
        justify-content: center;
    }

    @keyframes glowing-border {
    0% {
        box-shadow: 0 0 5px rgba(255, 255, 255, 0.2), 0 0 10px rgba(255, 255, 255, 0.2), 0 0 15px currentColor;
    }
    50% {
        box-shadow: 0 0 10px rgba(255, 255, 255, 0.4), 0 0 20px rgba(255, 255, 255, 0.4), 0 0 30px currentColor;
    }
    100% {
        box-shadow: 0 0 5px rgba(255, 255, 255, 0.2), 0 0 10px rgba(255, 255, 255, 0.2), 0 0 15px currentColor;
    }
}
    .reproductor-item {
        background-color: white;
        border-radius: 12px;
        padding: 20px 15px;
        box-shadow: 0 4px 15px rgba(223, 61, 158, 0.08);
        text-align: center;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: space-between;
        min-height: 250px;
        transition: transform 0.3s ease, box-shadow 0.3s ease;
        border: 2px solid #85d7ff;
        border-radius: 15px;
        box-shadow: 0 4px 15px rgba(223, 61, 158, 0.08);
        transition: border-color 0.3s ease, box-shadow 0.3s ease;
    }
    .reproductor-item:hover {
        border-color: #0099ff;
        box-shadow: 0 8px 25px rgba(77, 97, 197, 0.15);
    }

    .reproductor-item .circulo-reproductor:active {
        box-shadow:
            0 0 25px currentColor,
            0 0 40px currentColor,
            0 0 60px currentColor,
            inset 0 0 10px rgba(255, 255, 255, 0.2);
        transform: scale(0.97);
    }
    .reproductor-item .circulo-reproductor {
        width: 120px;
        height: 120px;
        border-radius: 50%;
        display: flex;
        align-items: center;
        justify-content: center;
        position: relative;
        overflow: hidden;
        margin-bottom: 15px;
        box-shadow: inset 0 0 10px rgba(19, 178, 175, 0.4);
        animation: glowing-border 3s infinite alternate ease-in-out;
        transition: box-shadow 0.3s ease-in-out;
    }

    .reproductor-item .circulo-reproductor:hover {
        box-shadow: 0 0 15px currentColor, 0 0 25px currentColor, 0 0 40px currentColor, inset 0 0 10px rgba(0, 0, 0, 0.1);
    }

    .reproductor-item .circulo-interno {
        width: 50px;
        height: 50px;
        border-radius: 50%;
        background-color: white;
        display: flex;
        align-items: center;
        justify-content: center;
        position: relative;
        overflow: hidden;
        z-index: 2;
    }

    .reproductor-item .wave-svg {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        display: block;
        z-index: 1;
        border-radius: 50%;
    }
    .reproductor-item .wave-svg path {
        transition: stroke 0.3s ease;
    }

    .reproductor-item .symbol-reproductor-side {
        position: absolute;
        width: 25px;
        height: 25px;
        object-fit: contain; 
        z-index: 1;
        top: 50%;
        transform: translateY(-50%);
        filter: brightness(0) invert(1); 
    }

    .reproductor-item .symbol-reproductor-left {
        left: 5px;
    }

    .reproductor-item .symbol-reproductor-right {
        right: 5px;
    }
  
    .reproductor-item p {
        font-size: 0.9em;
        color: #666;
        margin-bottom: 3px;
    }

    /* Nuevos estilos para el nombre de canción y usuario */
    .reproductor-item .song-name-display {
        font-size: 1.1em; 
        font-weight: bold; 
        color: #003058; 
        margin-bottom: 3px; 
        margin-top: -2px;
        padding-top: 1px;
    }
    .reproductor-item .user-name-display {
        font-size: 0.95em; /* Ligeramente más pequeño que la canción */
        color: #444; 
        margin-top: 5px; /* Espacio arriba */
        margin-bottom: 10px;
    }


    .reproductor-item .mood-display {
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 5px;
        font-size: 0.9em;
        color: #666;
        margin-bottom: 3px;
    }

    @media (max-width: 768px) {
        .form-crear-reproductor {
            grid-template-columns: 1fr;
        }
        .reproductor-compartido-container {
            padding: 20px;
            margin: 20px auto;
        }
    }
    .symbol-control2 {
        position: absolute;
        bottom: 15%;
        left: 47%;
        transform: translate(-80%, 40%);
        width: 20px;
        height: 20px;
        filter: brightness(0) invert(1);
    }


    .symbol-control1 {
        position: absolute;
        bottom: 15%;
        left: 60%;
        transform: translate(-50%, 40%);
        width: 20px;
        height: 20px;
        filter: brightness(0) invert(1);
    }
</style>

<div class="reproductor-compartido-container">
    <h2>Crea y Comparte tu Reproductor Musical</h2>
    <p>¡Arma tu propio reproductor con tu canción favorita y compártelo con el mundo!</p>

    <div class="form-crear-reproductor">
        <div class="form-group">
            <label for="userName">Tu Nombre / Apodo:</label>
            <input type="text" id="userName" bind:value={userName} />
        </div>

        <div class="form-group">
            <label for="songName">Nombre de la Canción:</label>
            <input
                type="text"
                id="songName"
                bind:value={songName}
                on:input={handleSongInput}
            />
            {#if songSuggestions.length > 0}
                <ul class="suggestions-list">
                    {#each songSuggestions as suggestion}
                        <!-- svelte-ignore a11y-click-events-have-key-events -->
                        <!-- svelte-ignore a11y-no-noninteractive-element-interactions -->
                        <li class="suggestion-item" on:click={() => selectSong(suggestion)}>
                            {suggestion}
                        </li>
                    {/each}
                </ul>
            {/if}
        </div>

        <div class="form-group">
            <label for="artistName">Artista:</label>
            <input
                type="text"
                id="artistName"
                bind:value={artistName}
                on:input={handleArtistInput}
            />
            {#if artistSuggestions.length > 0}
                <ul class="suggestions-list">
                    {#each artistSuggestions as suggestion}
                        <!-- svelte-ignore a11y-click-events-have-key-events -->
                        <!-- svelte-ignore a11y-no-noninteractive-element-interactions -->
                        <li class="suggestion-item" on:click={() => selectArtist(suggestion)}>
                            {suggestion}
                        </li>
                    {/each}
                </ul>
            {/if}
        </div>

        <div class="form-group">
            <label for="selectedGenre">Género Musical:</label>
            <input
                type="text"
                id="selectedGenre"
                bind:value={selectedGenre}
                on:input={handleGenreInput}
            />
            {#if genreSuggestions.length > 0}
                <ul class="suggestions-list">
                    {#each genreSuggestions as suggestion}
                        <!-- svelte-ignore a11y-click-events-have-key-events -->
                        <!-- svelte-ignore a11y-no-noninteractive-element-interactions -->
                        <li class="suggestion-item" on:click={() => selectGenre(suggestion)}>
                            {suggestion}
                        </li>
                    {/each}
                </ul>
            {/if}
            <div class="color-input-wrapper">
                <input type="color" id="selectedColor" bind:value={selectedColor} />
                <div class="form-group">
                    <label for="selectedColor">Color del Género:</label>
                    
                        <span class="color-description">Puedes personalizar el color que te guste!</span>
                
                </div>
            </div>
        </div>

        <div class="form-group">
            <label for="danzabilityInput">Danzabilidad (0-100):</label>
            <input
                type="number"
                id="danzabilityInput"
                bind:value={danzabilityInput}
                min="0"
                max="100"
            />
        </div>

        <div class="form-group">
            <label for="moodInput">Mood / Estado de Ánimo:</label>
            <input
                type="text"
                id="moodInput"
                bind:value={moodInput}
            />
        </div>

        <div class="form-group">
            {#if isCalculatingSymbol && !hasPlayed} <p>Tu Símbolo asignado: <span>Calculando...</span></p>
            {:else if assignedSymbol}
                <div class="symbol-display-container">
                    <p>Tu símbolo asignado: <span class="bold-symbol-text">{assignedSymbolText}</span></p>
                    <img src={assignedSymbol} alt="Símbolo asignado" class="assigned-symbol-preview" />
                </div>
                <p style="white-space: pre-line;">{assignedSymbolDescription}</p>
            {:else}
                <p>Tu símbolo asignado:</p>
                <p class="description-symbol">Aún no se ha calculado tu afinidad. ¡Juega para descubrirla!</p>
            {/if}
        </div>

        <button class="btn-crear-reproductor" on:click={createReproducer}>
            ¡Crear Mi Reproductor!
        </button>
    </div>

    <div class="reproductores-guardados">
        <h3>Reproductores Creados por la Comunidad:</h3>
        {#if $userReproducers.length === 0}
            <p style="text-align:center;">¡Sé el primero en crear un reproductor personalizado!</p>
        {:else}
            <div class="lista-reproductores">
                {#each $userReproducers as reproducer (reproducer.id)}
                    <div class="reproductor-item">
                        <p class="song-name-display">{reproducer.songName}</p>

                        <div
                            class="circulo-reproductor"
                            style="background-color: {reproducer.color};
                                    width: {obtenerDiametroReproductorUsuario()}px;
                                    height: {obtenerDiametroReproductorUsuario()}px;"
                        >
                            <img
                                src={reproducer.symbol}
                                alt="{reproducer.symbolText}"
                                class="symbol-reproductor-side symbol-reproductor-left"
                            />
                            <img
                                src={reproducer.symbol}
                                alt="{reproducer.symbolText}"
                                class="symbol-reproductor-side symbol-reproductor-right"
                            />

                            <img src="/images/Pause.png" alt="Pause" class="symbol-control1" />
                            <img src="/images/Play.png" alt="Play" class="symbol-control2" />

                            <div class="circulo-interno">
                                <svg
                                    class="wave-svg"
                                    width="100%"
                                    height="100%"
                                    viewBox="0 0 100 30"
                                    preserveAspectRatio="none"
                                >
                                    <path
                                        d={wavePathReproductorUsuario(reproducer.danzability)}
                                        fill="none"
                                        stroke={reproducer.color}
                                        stroke-width="{strokeWidthReproductorUsuario()}"
                                    />
                                </svg>
                            </div>
                        </div>

                        <p class="user-name-display">{reproducer.userName}</p>

                        <p>{reproducer.artistName}</p>
                        <p>{reproducer.genre}</p>
                        <p>Danzabilidad: {reproducer.danzability}%</p>
                        <p class="mood-display">
                            Mood: {getMoodEmoji(reproducer.mood)} {reproducer.mood}
                        </p>
                    </div>
                {/each}
            </div>
                  
            <button class="btn-clear-reproducers" on:click={clearReproducers}>
                Borrar Todos los Reproductores
            </button>
        {/if}
    </div>
</div>
