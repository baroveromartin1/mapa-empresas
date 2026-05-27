cat << 'HTMLEOF' > /mnt/user-data/outputs/index.html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Mapa de Empresas — Argentina</title>
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@tabler/icons-webfont@latest/tabler-icons.min.css"/>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; background: #f4f6f9; color: #2c3e50; height: 100vh; display: flex; flex-direction: column; }
    header { background: #1a2740; color: #fff; padding: 12px 20px; display: flex; align-items: center; gap: 12px; flex-shrink: 0; }
    header h1 { font-size: 16px; font-weight: 500; }
    header span { font-size: 12px; color: #8a99b0; margin-left: auto; }
    .container { display: flex; gap: 12px; padding: 12px; flex: 1; overflow: hidden; }
    #map { flex: 1; border-radius: 10px; border: 1px solid #dce3ed; z-index: 0; }
    .panel { width: 270px; display: flex; flex-direction: column; gap: 10px; overflow: hidden; }
    .card { background: #fff; border: 1px solid #dce3ed; border-radius: 10px; padding: 14px; }
    .card-title { font-weight: 500; font-size: 13px; color: #1a2740; margin-bottom: 10px; display: flex; align-items: center; gap: 6px; }
    input[type=text] { width: 100%; padding: 7px 10px; border: 1px solid #dce3ed; border-radius: 7px; font-size: 13px; color: #2c3e50; background: #f9fafc; outline: none; margin-bottom: 7px; }
    input[type=text]:focus { border-color: #378ADD; background: #fff; }
    .btn-row { display: flex; gap: 6px; }
    button { padding: 7px 12px; border: 1px solid #dce3ed; border-radius: 7px; background: #fff; color: #2c3e50; font-size: 12px; cursor: pointer; display: flex; align-items: center; gap: 5px; transition: background 0.15s; }
    button:hover { background: #f0f4f8; }
    button.primary { background: #1a2740; color: #fff; border-color: #1a2740; flex: 1; justify-content: center; }
    button.primary:hover { background: #243655; }
    button.secondary { flex: 1; justify-content: center; }
    #coord-info { font-size: 11px; color: #8a99b0; margin-top: 5px; min-height: 15px; }
    .list-card { flex: 1; display: flex; flex-direction: column; overflow: hidden; }
    #lista-empresas { flex: 1; overflow-y: auto; display: flex; flex-direction: column; gap: 6px; margin-top: 6px; }
    .empresa-item { background: #f9fafc; border: 1px solid #dce3ed; border-radius: 8px; padding: 8px 10px; }
    .empresa-item .nombre { font-weight: 500; font-size: 12px; color: #1a2740; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
    .empresa-item .contacto { font-size: 11px; color: #5a6a85; margin-top: 2px; }
    .empresa-item .ciudad { font-size: 10px; color: #8a99b0; margin-top: 1px; }
    .empresa-item .acciones { display: flex; gap: 3px; margin-top: 6px; }
    .empresa-item .acciones button { padding: 3px 7px; font-size: 11px; }
    .empresa-item .acciones .btn-del { color: #c0392b; border-color: #f5c6c6; }
    .empresa-item .acciones .btn-del:hover { background: #fdf0f0; }
    #contador { font-size: 11px; color: #8a99b0; margin-left: auto; }
    .click-hint { position: absolute; bottom: 20px; left: 50%; transform: translateX(-50%); background: rgba(0,0,0,0.6); color: #fff; padding: 6px 16px; border-radius: 20px; font-size: 12px; pointer-events: none; z-index: 999; white-space: nowrap; }
    .empty-msg { font-size: 12px; color: #8a99b0; text-align: center; padding: 20px 0; }
    #btn-guardar { display: none; }
    .leaflet-popup-content b { color: #1a2740; }
  </style>
</head>
<body>

<header>
  <i class="ti ti-map-pin" style="font-size:20px;"></i>
  <h1>Mapa de Empresas — Argentina</h1>
  <span id="header-count">0 empresas</span>
</header>

<div class="container">
  <div style="position:relative;flex:1;display:flex;">
    <div id="map"></div>
    <div class="click-hint" id="click-hint">Hacé clic en el mapa para agregar una empresa</div>
  </div>

  <div class="panel">
    <div class="card">
      <div class="card-title"><i class="ti ti-building"></i> Nueva empresa</div>
      <input type="text" id="inp-nombre" placeholder="Nombre de la empresa" />
      <input type="text" id="inp-contacto" placeholder="Contacto (tel / email)" />
      <input type="text" id="inp-ciudad" placeholder="Ciudad o dirección" />
      <div class="btn-row">
        <button class="secondary" onclick="buscarDireccion()"><i class="ti ti-search"></i> Buscar</button>
        <button class="primary" id="btn-guardar" onclick="guardarEmpresa()"><i class="ti ti-check"></i> Guardar</button>
      </div>
      <div id="coord-info"></div>
    </div>

    <div class="card list-card">
      <div class="card-title">
        <i class="ti ti-list"></i> Empresas
        <span id="contador">0</span>
      </div>
      <input type="text" id="inp-filtro" placeholder="Filtrar empresas..." oninput="renderLista()" style="margin-bottom:0;" />
      <div id="lista-empresas">
        <div class="empty-msg">Sin empresas aún</div>
      </div>
    </div>
  </div>
</div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script>
const STORAGE_KEY = 'mapa-empresas-arg';
let map, tempMarker, pendingLatLng, editandoId = null;
let empresas = [];

function initMap() {
  map = L.map('map').setView([-38.5, -63.5], 4);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© <a href="https://openstreetmap.org">OpenStreetMap</a>',
    maxZoom: 18
  }).addTo(map);
  map.on('click', function(e) {
    setPendingLocation(e.latlng.lat, e.latlng.lng);
    document.getElementById('click-hint').style.display = 'none';
  });
}

function setPendingLocation(lat, lng) {
  if (tempMarker) map.removeLayer(tempMarker);
  pendingLatLng = { lat, lng };
  const icon = L.divIcon({
    className: '',
    html: '<div style="width:20px;height:20px;border-radius:50%;background:#378ADD;border:3px solid #fff;box-shadow:0 2px 6px rgba(0,0,0,0.4);"></div>',
    iconAnchor: [10, 10]
  });
  tempMarker = L.marker([lat, lng], { icon }).addTo(map);
  document.getElementById('coord-info').textContent = lat.toFixed(5) + ', ' + lng.toFixed(5);
  document.getElementById('btn-guardar').style.display = 'flex';
}

async function buscarDireccion() {
  const q = document.getElementById('inp-ciudad').value.trim();
  if (!q) return;
  document.getElementById('coord-info').textContent = 'Buscando...';
  try {
    const r = await fetch(`https://nominatim.openstreetmap.org/search?format=json&countrycodes=ar&q=${encodeURIComponent(q)}&limit=1`);
    const data = await r.json();
    if (data.length) {
      const { lat, lon } = data[0];
      map.setView([+lat, +lon], 13);
      setPendingLocation(+lat, +lon);
    } else {
      document.getElementById('coord-info').textContent = 'No se encontró la ubicación';
    }
  } catch(e) {
    document.getElementById('coord-info').textContent = 'Error de conexión';
  }
}

function guardarEmpresa() {
  const nombre = document.getElementById('inp-nombre').value.trim();
  const contacto = document.getElementById('inp-contacto').value.trim();
  const ciudad = document.getElementById('inp-ciudad').value.trim();
  if (!nombre) { document.getElementById('coord-info').textContent = '⚠ Ingresá el nombre'; return; }
  if (!pendingLatLng) { document.getElementById('coord-info').textContent = '⚠ Seleccioná una ubicación'; return; }

  if (editandoId !== null) {
    const idx = empresas.findIndex(e => e.id === editandoId);
    if (idx !== -1) { map.removeLayer(empresas[idx].marker); empresas.splice(idx, 1); }
    editandoId = null;
  }

  const id = Date.now();
  const marker = crearMarker(pendingLatLng.lat, pendingLatLng.lng, nombre, contacto, ciudad, id);
  empresas.push({ id, nombre, contacto, ciudad, lat: pendingLatLng.lat, lng: pendingLatLng.lng, marker });
  guardarLocal();
  renderLista();
  limpiarForm();
}

function crearMarker(lat, lng, nombre, contacto, ciudad, id) {
  const icon = L.divIcon({
    className: '',
    html: `<div style="width:22px;height:22px;border-radius:50%;background:#1D9E75;border:3px solid #fff;box-shadow:0 2px 6px rgba(0,0,0,0.35);cursor:pointer;"></div>`,
    iconAnchor: [11, 11]
  });
  const m = L.marker([lat, lng], { icon }).addTo(map);
  m.bindPopup(`<b>${nombre}</b><br/><span style="color:#666;font-size:12px;">${contacto || '—'}</span><br/><span style="color:#999;font-size:11px;">${ciudad || ''}</span>`);
  return m;
}

function limpiarForm() {
  ['inp-nombre','inp-contacto','inp-ciudad'].forEach(id => document.getElementById(id).value = '');
  document.getElementById('coord-info').textContent = '';
  document.getElementById('btn-guardar').style.display = 'none';
  if (tempMarker) { map.removeLayer(tempMarker); tempMarker = null; }
  pendingLatLng = null;
  editandoId = null;
}

function renderLista() {
  const filtro = document.getElementById('inp-filtro').value.toLowerCase();
  const lista = document.getElementById('lista-empresas');
  const filtradas = empresas.filter(e =>
    e.nombre.toLowerCase().includes(filtro) ||
    (e.contacto && e.contacto.toLowerCase().includes(filtro)) ||
    (e.ciudad && e.ciudad.toLowerCase().includes(filtro))
  );
  document.getElementById('contador').textContent = empresas.length;
  document.getElementById('header-count').textContent = empresas.length + ' empresa' + (empresas.length !== 1 ? 's' : '');
  if (!filtradas.length) {
    lista.innerHTML = '<div class="empty-msg">Sin resultados</div>';
    return;
  }
  lista.innerHTML = filtradas.map(e => `
    <div class="empresa-item">
      <div class="nombre">${e.nombre}</div>
      ${e.contacto ? `<div class="contacto"><i class="ti ti-phone" style="font-size:11px;"></i> ${e.contacto}</div>` : ''}
      ${e.ciudad ? `<div class="ciudad"><i class="ti ti-map-pin" style="font-size:10px;"></i> ${e.ciudad}</div>` : ''}
      <div class="acciones">
        <button onclick="centrarEn(${e.id})" title="Ver en mapa"><i class="ti ti-map-pin"></i> Ver</button>
        <button onclick="editarEmpresa(${e.id})" title="Editar"><i class="ti ti-edit"></i> Editar</button>
        <button class="btn-del" onclick="eliminarEmpresa(${e.id})" title="Eliminar"><i class="ti ti-trash"></i></button>
      </div>
    </div>
  `).join('');
}

function centrarEn(id) {
  const e = empresas.find(e => e.id === id);
  if (!e) return;
  map.setView([e.lat, e.lng], 13);
  e.marker.openPopup();
}

function editarEmpresa(id) {
  const e = empresas.find(e => e.id === id);
  if (!e) return;
  editandoId = id;
  document.getElementById('inp-nombre').value = e.nombre;
  document.getElementById('inp-contacto').value = e.contacto || '';
  document.getElementById('inp-ciudad').value = e.ciudad || '';
  setPendingLocation(e.lat, e.lng);
  document.getElementById('coord-info').textContent = 'Editando — podés mover el pin';
}

function eliminarEmpresa(id) {
  if (!confirm('¿Eliminar esta empresa?')) return;
  const idx = empresas.findIndex(e => e.id === id);
  if (idx === -1) return;
  map.removeLayer(empresas[idx].marker);
  empresas.splice(idx, 1);
  guardarLocal();
  renderLista();
}

function guardarLocal() {
  const datos = empresas.map(({ id, nombre, contacto, ciudad, lat, lng }) => ({ id, nombre, contacto, ciudad, lat, lng }));
  localStorage.setItem(STORAGE_KEY, JSON.stringify(datos));
}

function cargarLocal() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return;
    const datos = JSON.parse(raw);
    datos.forEach(e => {
      const marker = crearMarker(e.lat, e.lng, e.nombre, e.contacto, e.ciudad, e.id);
      empresas.push({ ...e, marker });
    });
    renderLista();
    if (empresas.length) document.getElementById('click-hint').style.display = 'none';
  } catch(e) {}
}

initMap();
cargarLocal();
</script>
</body>
</html>
HTMLEOF
echo "OK"
