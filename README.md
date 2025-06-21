<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <title>Match Fútbol</title>
  <meta name="description" content="Encuentra y crea partidos de fútbol cerca de ti en tiempo real. Match Fútbol conecta jugadores en tu ciudad.">

  <title>Match Fútbol</title>

  <script src="https://cdn.tailwindcss.com"></script>
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.11.0/firebase-app.js";
    import { getDatabase, ref, onChildAdded, push, set, remove, onValue, off } from "https://www.gstatic.com/firebasejs/10.11.0/firebase-database.js";

    const firebaseConfig = {
      apiKey: "AIzaSyA3WTlBtO2nd841K0fkPfYT0NQRMdQDl1k",
      authDomain: "match-futbol-dc63b.firebaseapp.com",
      projectId: "match-futbol-dc63b",
      storageBucket: "match-futbol-dc63b.appspot.com",
      messagingSenderId: "1030565365931",
      appId: "1:1030565365931:web:b3ea0a6249e7b7c9464fee",
      measurementId: "G-K4VZ928FN2"
    };

    const app = initializeApp(firebaseConfig);
    const db = getDatabase(app);

    let map, miRef = null, miId = null, tipoActual = null, chatEscuchado = null;
    let miMarker = null;
    const iconBlue = new L.Icon({ iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-blue.png', shadowUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png', iconSize: [25,41], iconAnchor: [12,41], popupAnchor: [1,-34], shadowSize: [41,41]});
    const iconRed = new L.Icon({ iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-red.png', shadowUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png', iconSize: [25,41], iconAnchor: [12,41], popupAnchor: [1,-34], shadowSize: [41,41]});

    window.onload = () => {
      document.getElementById("crearBtn").onclick = () => mostrarCrearForm();
      document.getElementById("buscarBtn").onclick = () => buscarPartidas();
      document.getElementById("formCrear").onsubmit = (e) => {
        e.preventDefault();
        crearPartida();
        document.getElementById("formCrear").classList.add("hidden");
      };
      initMapa();
    };

    function initMapa(){
      navigator.geolocation.getCurrentPosition((pos) => {
        const lat = pos.coords.latitude, lon = pos.coords.longitude;
        map = L.map('mapa').setView([lat,lon],15);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
      }, () => alert("Activa GPS y recarga la página."));
    }

    function mostrarCrearForm(){
      document.getElementById("formCrear").classList.remove("hidden");
    }

    function limpiarMapa(){
      Object.values(map._layers).forEach(l => {
        if (l instanceof L.Marker) map.removeLayer(l);
      });
      if(miMarker){ map.removeLayer(miMarker); miMarker = null; }
    }

    function buscarPartidas(){
      limpiarMapa();
      tipoActual = "buscar";
      if(chatEscuchado) chatEscuchado.off();
      onChildAdded(ref(db,"partidas"), data=> {
        const val = data.val();
        L.marker([val.lat,val.lon],{ icon: iconBlue }).addTo(map)
          .bindPopup(`<b>Fútbol ${val.tipoFutbol}</b><br>Suplentes: ${val.suplentes}`);
      });
    }

    function crearPartida(){
      limpiarMapa();
      tipoActual = "crear";
      if(miRef) remove(miRef);
      navigator.geolocation.getCurrentPosition((pos)=>{
        const lat=pos.coords.latitude, lon=pos.coords.longitude;
        const nr = push(ref(db,"partidas"));
        miRef = nr; miId = nr.key;
        set(nr, { lat, lon, tipo:"crear", tipoFutbol:document.getElementById("tipoFutbol").value, suplentes:document.getElementById("suplentes").value });
        window.addEventListener("beforeunload",()=>remove(miRef));
        miMarker = L.marker([lat,lon],{icon:iconBlue}).addTo(map).bindPopup("Tu partida").openPopup();
      });
    }

    window.iniciarChat = function(destinoId, miTipo){
      if((miTipo==="crear"&&destinoId.startsWith("crear")) || (miTipo==="buscar"&&destinoId.startsWith("buscar"))){
        alert("Solo puedes chatear con jugador del otro rol."); return;
      }
      document.getElementById("chatLateral").classList.remove("hidden");
      const cb = document.getElementById("chatBox"), ci = document.getElementById("chatInput");
      cb.innerHTML="";
      const chatId = [miId,destinoId].sort().join("_");
      const cr = ref(db,`chats/${chatId}`);
      chatEscuchado = onValue(cr,snap=>{
        cb.innerHTML="";
        snap.forEach(msj=>{
          const v=msj.val(), hora=new Date(v.ts).toLocaleTimeString();
          const lado = v.uid===miId?"text-right text-green-700":"text-left text-gray-700";
          cb.innerHTML += `<p class="${lado}"><b>${v.uid===miId?"Tú":"Jugador"}:</b> ${v.texto}<br><small>${hora}</small></p>`;
        });
        cb.scrollTop=cb.scrollHeight;
      });
      document.getElementById("btnEnviar").onclick=()=>{
        const txt=ci.value.trim(); if(txt){
          push(cr,{ uid:miId, texto:txt, ts:Date.now() });
          ci.value="";
        }
      };
    };

    window.cerrarChat = ()=> {
      document.getElementById("chatLateral").classList.add("hidden");
      if(chatEscuchado) chatEscuchado.off();
    };

  </script>
</head>
<body class="bg-gray-100 p-4">

  <div class="bg-white shadow-lg rounded-2xl p-6 max-w-md mx-auto text-center">
    <h1 class="text-3xl font-bold text-green-600 mb-6">Match Fútbol</h1>
    <button id="crearBtn" class="w-full mb-4 bg-green-500 text-white py-2 rounded hover:bg-green-600">➕ Crear Partida</button>
    <button id="buscarBtn" class="w-full bg-blue-500 text-white py-2 rounded hover:bg-blue-600">🔍 Buscar Partidas</button>

    <form id="formCrear" class="mt-4 hidden">
      <label class="block text-left mb-2">Tipo de fútbol:</label>
      <select id="tipoFutbol" class="w-full mb-4 border p-1">
        <option value="5">5</option><option value="6">6</option><option value="7">7</option><option value="8">8</option><option value="9">9</option><option value="10">10</option><option value="11">11</option>
      </select>
      <label class="block text-left mb-2">¿Con suplentes?</label>
      <select id="suplentes" class="w-full mb-4 border p-1">
        <option>Sí</option><option>No</option>
      </select>
      <button type="submit" class="w-full bg-green-600 text-white py-2 rounded">Confirmar</button>
    </form>
  </div>

  <div id="mapa" class="h-80 max-w-4xl mx-auto mt-6 rounded shadow"></div>

  <div id="chatLateral" class="fixed right-4 bottom-4 w-80 bg-white p-4 rounded shadow hidden">
    <div><h2 class="font-bold mb-2">Chat</h2></div>
    <div id="chatBox" class="h-40 overflow-y-auto mb-2 bg-gray-100 p-2 text-sm"></div>
    <div class="flex gap-2">
      <input id="chatInput" class="border p-1 flex-grow" placeholder="Mensaje..."/>
      <button id="btnEnviar" class="bg-green-500 text-white px-3 rounded">Enviar</button>
    </div>
    <button onclick="cerrarChat()" class="mt-2 text-sm text-gray-600">Cerrar</button>
  </div>

</body>
</html>
[Uploadgoogle-site-verification: googlede5498f64984b819.htmling googlede5498f64984b819.html…]()

