
[velobit_new.py](https://github.com/user-attachments/files/31101895/velobit_new.py)
from flask import Flask, render_template_string, request, jsonify
import sqlite3
import math
import json
 
print("Файл запущен!")
app = Flask(__name__)
DATABASE = 'trails.db'
ADMIN_PASSWORD = 'velobit2026'

def haversine(lat1, lon1, lat2, lon2):
    R = 6371
    dlat = math.radians(lat2 - lat1)
    dlon = math.radians(lon2 - lon1)
    a = math.sin(dlat/2)**2 + math.cos(math.radians(lat1)) * math.cos(math.radians(lat2)) * math.sin(dlon/2)**2
    c = 2 * math.atan2(math.sqrt(a), math.sqrt(1-a))
    return R * c


    def init_db():
        with sqlite3.connect(DATABASE) as conn:
           conn.execute('''CREATE TABLE IF NOT EXISTS trails (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            city TEXT DEFAULT 'Челябинск',
            level TEXT NOT NULL,
            description TEXT,
            image_url TEXT,
            rating REAL DEFAULT 0,
            votes INTEGER DEFAULT 0,
            coordinates TEXT DEFAULT '[]'
        )''')
        
        conn.execute('''CREATE TABLE IF NOT EXISTS partners (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            contact_person TEXT DEFAULT '',
            phone TEXT DEFAULT '',
            email TEXT DEFAULT '',
            city TEXT DEFAULT 'Челябинск',
            status TEXT DEFAULT 'Лид',
            price REAL DEFAULT 5000,
            paid_until TEXT DEFAULT '',
            notes TEXT DEFAULT '',
            lat REAL DEFAULT 0,
            lon REAL DEFAULT 0,
            address TEXT DEFAULT '',
            discount_code TEXT DEFAULT 'VELO10',
            discount_text TEXT DEFAULT 'Скидка 10%'
        )''')
        
        conn.execute('''CREATE TABLE IF NOT EXISTS partner_notes (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            partner_id INTEGER,
            note TEXT,
            created_at TEXT DEFAULT (datetime('now'))
        )''')
        
        conn.execute('''CREATE TABLE IF NOT EXISTS reviews (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            trail_id INTEGER,
            user_name TEXT DEFAULT 'Аноним',
            rating INTEGER CHECK(rating >= 1 AND rating <= 5),
            comment TEXT,
            created_at TEXT DEFAULT (datetime('now')),
            FOREIGN KEY(trail_id) REFERENCES trails(id) ON DELETE CASCADE
        )''')
        
        conn.execute('''CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT NOT NULL,
            total_rides INTEGER DEFAULT 0,
            total_km REAL DEFAULT 0
        )''')
        
        if not conn.execute("SELECT COUNT(*) FROM trails").fetchone()[0]:
            conn.executemany("INSERT INTO trails (name, city, level, description, image_url, coordinates) VALUES (?, ?, ?, ?, ?, ?)", [
                ("Городской Бор", "Челябинск", "Синий", "Грунт, корни, перепад высот.", "", "[[55.1750,61.3800],[55.1800,61.3880],[55.1840,61.3950]]"),
                ("Шершнёвский карьер", "Челябинск", "Красный", "Камни, резкие спуски, техничный рельеф.", "", "[[55.1200,61.4200],[55.1250,61.4300],[55.1280,61.4360]]"),
                ("Тополиная аллея", "Челябинск", "Зелёный", "Спокойный маршрут вдоль аллеи.", "", "[[55.2000,61.4700],[55.2050,61.4800]]"),
                ("Каштакский бор", "Челябинск", "Синий", "Сосновый лес, песчаные тропы.", "", "[[55.2100,61.5000],[55.2150,61.5100],[55.2100,61.5000]]")
            ])
        
        if not conn.execute("SELECT COUNT(*) FROM partners").fetchone()[0]:
            conn.executemany("INSERT INTO partners (name, contact_person, phone, email, city, status, price, lat, lon, address) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)", [
                ("Триал-Спорт", "Алексей", "+79000000001", "trial74@mail.ru", "Челябинск", "Лид", 5000, 55.1644, 61.4368, "ул. Кирова, 15"),
                ("Веломастерская Шершни", "Олег", "+79000000002", "shershni@mail.ru", "Челябинск", "Лид", 3000, 55.1500, 61.4200, "ул. Молодогвардейцев, 10")
            ])
        
        if not conn.execute("SELECT COUNT(*) FROM users").fetchone()[0]:
            conn.executemany("INSERT INTO users (username, total_rides, total_km) VALUES (?, ?, ?)", [
                ("Алексей Воронов", 89, 1250),
                ("Екатерина Смирнова", 67, 980),
                ("Дмитрий Петров", 54, 870),
                ("Анна Ковалёва", 48, 760),
                ("Иван Соколов", 41, 650),
                ("Мария Попова", 35, 540),
                ("Сергей Васильев", 28, 430),
                ("Ольга Новикова", 22, 320),
                ("Павел Морозов", 15, 210),
                ("Татьяна Кузнецова", 10, 150)
            ])
HTML = """
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>🚲 Velobit</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #F4F4F9; font-family: sans-serif; display: flex; height: 100vh; overflow: hidden; }
        .sidebar { width: 220px; background: rgba(255,255,255,0.85); padding: 16px; overflow-y: auto; border-right: 1px solid #ddd; flex-shrink: 0; }
        .sidebar h1 { font-size: 20px; color: #10B981; }
        .sidebar input, .sidebar select { width: 100%; padding: 8px; margin: 4px 0; border: 1px solid #ddd; border-radius: 6px; }
        .sidebar button { width: 100%; padding: 8px; border: none; border-radius: 6px; cursor: pointer; font-weight: 600; }
        .btn-primary { background: #3B82F6; color: white; }
        .btn-green { background: #10B981; color: white; }
        .btn-orange { background: #F59E0B; color: white; }
        .btn-dark { background: #1F2937; color: white; }
        .btn-group { display: flex; gap: 6px; margin: 10px 0; }
        .btn-group button { flex: 1; }
        .theme-btn { background: none; border: none; font-size: 20px; cursor: pointer; float: right; }
        .map-container { flex: 1; position: relative; height: 100vh; }
        #map { height: 100%; width: 100%; }
        body.dark #map { filter: brightness(0.6) saturate(0.7); }
        body.dark { background: #0B0E14; color: #fff; }
        body.dark .sidebar { background: rgba(20,20,30,0.85); border-color: #333; }
        body.dark .sidebar input, body.dark .sidebar select { background: #222; color: #fff; border-color: #444; }
        .counter { position: absolute; top: 20px; right: 20px; background: white; padding: 6px 14px; border-radius: 40px; box-shadow: 0 2px 15px rgba(0,0,0,0.08); z-index: 1000; }
        .right-panel { width: 280px; background: rgba(255,255,255,0.85); padding: 16px; overflow-y: auto; border-left: 1px solid #ddd; flex-shrink: 0; }
        .right-panel h3 { color: #F59E0B; margin-bottom: 12px; }
        .trail-card { background: white; padding: 12px 14px; margin-bottom: 6px; border-radius: 10px; cursor: pointer; border-left: 4px solid #10B981; box-shadow: 0 2px 8px rgba(0,0,0,0.06); }
        .trail-card:hover { transform: translateX(4px); }
        .trail-card h4 { font-size: 13px; font-weight: 700; }
        .trail-card p { font-size: 11px; color: #555; }
        .add-info { display: none; margin-top: 10px; padding: 10px; background: #10B981; color: white; border-radius: 8px; font-size: 13px; }
        .add-info button { margin-top: 4px; padding: 6px; border: none; border-radius: 6px; cursor: pointer; font-weight: 600; }
        .add-info .btn-white { background: white; color: #10B981; }
        .trail-info-panel { background: rgba(255,255,255,0.9); border-radius: 10px; padding: 12px; margin-top: 12px; border-left: 4px solid #10B981; display: none; }
        .trail-info-panel h4 { color: #10B981; margin-bottom: 4px; font-size: 14px; }
        .trail-info-panel p { font-size: 12px; margin: 2px 0; color: #555; }
        .tag { display: inline-block; padding: 2px 10px; border-radius: 20px; font-size: 11px; font-weight: 600; }
        .tag-green { background: #10B98120; color: #10B981; }
        .tag-blue { background: #3B82F620; color: #3B82F6; }
        .tag-red { background: #EF444420; color: #EF4444; }
        .modal-overlay { display: none; position: fixed; inset: 0; background: rgba(0,0,0,0.5); backdrop-filter: blur(4px); z-index: 9999; align-items: center; justify-content: center; }
        .modal-overlay.show { display: flex; }
        .modal { background: white; border-radius: 20px; padding: 30px; max-width: 500px; width: 90%; max-height: 80vh; overflow-y: auto; box-shadow: 0 20px 60px rgba(0,0,0,0.3); }
        .modal .close-btn { float: right; background: none; border: none; font-size: 24px; cursor: pointer; }
        .review-item { padding: 10px 0; border-bottom: 1px solid #eee; }
        .review-item .review-stars { color: #f59e0b; font-size: 16px; }
        .review-item .review-name { font-weight: 600; }
        .star { font-size: 24px; cursor: pointer; color: #d1d5db; }
        .star.active { color: #f59e0b; }
        .review-form textarea { width: 100%; padding: 10px; border: 1px solid #ddd; border-radius: 10px; resize: vertical; margin-top: 8px; }
        .review-form input { width: 100%; padding: 10px; border: 1px solid #ddd; border-radius: 10px; margin-bottom: 8px; }
        .review-form button { width: 100%; padding: 10px; background: #10B981; color: white; border: none; border-radius: 10px; font-weight: 600; cursor: pointer; margin-top: 8px; }
        .calc-block { border-top: 1px solid #ddd; padding-top: 12px; margin-top: 12px; }
        .calc-block h3 { font-size: 14px; color: #F59E0B; }
        @media (max-width: 768px) {
            body { flex-direction: column; }
            .sidebar { width: 100%; height: 200px; border-right: none; border-bottom: 1px solid #ddd; }
            .right-panel { width: 100%; height: 40vh; border-left: none; border-top: 1px solid #ddd; }
            .map-container { height: calc(100vh - 200px - 40vh); }
        }
    </style>
</head>
<body>

<div class="sidebar">
    <h1>🚲 VELOBIT</h1>
    <button class="theme-btn" onclick="toggleTheme()">🌓</button>
    <div style="margin:12px 0;">
        <input type="text" id="placeSearch" placeholder="📍 Поиск места">
        <button class="btn-primary" onclick="searchPlace()">🔍 Найти место</button>
        <input type="text" id="searchInput" placeholder="🔍 Поиск трейла...">
        <select id="levelFilter">
            <option value="Все">Все уровни</option>
            <option value="Зелёный">🟢 Зелёный</option>
            <option value="Синий">🔵 Синий</option>
            <option value="Красный">🔴 Красный</option>
            <option value="Чёрный">⚫ Чёрный</option>
        </select>
    </div>
    <div class="btn-group">
        <button class="btn-green" onclick="toggleAddMode()">➕ Добавить</button>
        <button id="editModeBtn" class="btn-primary" onclick="toggleEditMode()">✏️ Правка</button>
    </div>
    <div id="addInfo" class="add-info">
        🖱️ Кликай по карте
        <button class="btn-white" onclick="finishTrail()">✅ Завершить</button>
        <button onclick="toggleAddMode()">❌ Отмена</button>
    </div>
    <div class="calc-block">
        <h3>⛰️ Калькулятор высоты</h3>
        <input type="number" id="elevDist" placeholder="Дистанция (км)">
        <input type="number" id="elevGrade" placeholder="Уклон (%)">
        <button class="btn-green" onclick="calcElev()">Рассчитать</button>
        <div id="elevResult" style="margin-top:6px;font-size:13px;"></div>
    </div>
    <div class="calc-block">
        <h3>🧮 Калькулятор калорий</h3>
        <input type="number" id="calcWeight" placeholder="Вес (кг)">
        <input type="number" id="calcDist" placeholder="Дистанция (км)">
        <input type="number" id="calcHours" placeholder="Время (часы)">
        <button class="btn-orange" onclick="calcCal()">Рассчитать</button>
        <div id="calcResult" style="margin-top:6px;font-size:13px;"></div>
    </div>
    <a href="/main" class="btn-dark" style="display:block;text-align:center;text-decoration:none;margin:8px 0;border-radius:6px;padding:8px;">🏠 Главная</a>
    <a href="/admin/crm?password=velobit2026" class="btn-dark" style="display:block;text-align:center;text-decoration:none;margin-top:12px;border-radius:6px;padding:8px;">🏪 CRM</a>
</div>

<div class="map-container">
    <div id="map"></div>
    <div class="counter">👁 <span id="viewCount">0</span></div>
</div>

<div class="right-panel">
    <h3>📍 Трейлы</h3>
    <div id="trailsList"></div>
    <div id="trailInfo" class="trail-info-panel">
        <h4 id="infoName">Название</h4>
        <p>📏 <span id="infoDist">0</span> км</p>
        <p>⏱️ <span id="infoTime">0</span> мин</p>
        <p>⛰️ <span id="infoLevel" class="tag tag-green">—</span></p>
        <p id="infoDesc" style="font-size:12px;color:#555;margin-top:4px;"></p>
    </div>
</div>

<!-- МОДАЛКА ОТЗЫВОВ -->
<div class="modal-overlay" id="reviewModal">
    <div class="modal">
        <button class="close-btn" onclick="closeModal()">✕</button>
        <h2>📝 Отзывы</h2>
        <div id="reviewsContent">
            <p style="color:#9CA3AF;">Загрузка...</p>
        </div>
        <div class="review-form">
            <h4 style="margin:16px 0 8px;">Оставить отзыв</h4>
            <input type="text" id="reviewName" placeholder="Ваше имя">
            <div id="starSelector">
                <span class="star" onclick="selStar(1)">☆</span>
                <span class="star" onclick="selStar(2)">☆</span>
                <span class="star" onclick="selStar(3)">☆</span>
                <span class="star" onclick="selStar(4)">☆</span>
                <span class="star" onclick="selStar(5)">☆</span>
            </div>
            <textarea id="reviewComment" rows="3" placeholder="Поделись впечатлениями..."></textarea>
            <button onclick="sendReview()" class="btn-green" style="width:100%;padding:10px;font-weight:600;cursor:pointer;">✅ Отправить</button>
        </div>
    </div>
</div>

<script>
var map = L.map('map').setView([55.1644, 61.4368], 12);
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {attribution:'OSM'}).addTo(map);
var allTrails = [], markers = L.layerGroup().addTo(map);
var addMode = false, newCoords = [], tempMarkers = [], tempLine = null;
var selRating = 0, curTrail = null;
var editMode = false;
var selectedTrailId = null;
var editMarkers = [];
var editPolyline = null;

function toggleTheme() {
    document.body.classList.toggle('dark');
    localStorage.setItem('theme', document.body.classList.contains('dark') ? 'dark' : 'light');
}
if (localStorage.getItem('theme') === 'dark') document.body.classList.add('dark');

function renderStars(r) {
    var s = '';
    for(var i=0;i<5;i++) s += i < Math.floor(r) ? '★' : (i < r ? '½' : '☆');
    return s;
}

function showInfo(t) {
    var c = JSON.parse(t.coordinates||'[]');
    var d = 0;
    for(var i=0;i<c.length-1;i++){
        var R=6371, lat1=c[i][0], lon1=c[i][1], lat2=c[i+1][0], lon2=c[i+1][1];
        var dLat=(lat2-lat1)*Math.PI/180, dLon=(lon2-lon1)*Math.PI/180;
        var a=Math.sin(dLat/2)**2 + Math.cos(lat1*Math.PI/180)*Math.cos(lat2*Math.PI/180)*Math.sin(dLon/2)**2;
        d += R*2*Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
    }
    d = d.toFixed(1);
    var tm = (d/15*60).toFixed(0);
    var cls = 'tag-green';
    if(t.level==='Синий') cls='tag-blue';
    else if(t.level==='Красный') cls='tag-red';
    else if(t.level==='Чёрный') cls='tag-black';
    document.getElementById('infoName').textContent = t.name;
    document.getElementById('infoDist').textContent = d+' км';
    document.getElementById('infoTime').textContent = tm+' мин';
    document.getElementById('infoLevel').textContent = t.level||'—';
    document.getElementById('infoLevel').className = 'tag ' + cls;
    document.getElementById('infoDesc').textContent = t.description||'Описание отсутствует';
    document.getElementById('trailInfo').style.display = 'block';
}

function loadTrails() {
    fetch('/api/trails/stats').then(r=>r.json()).then(d=>{ allTrails=d; renderTrails(); });
}

function renderTrails() {
    var f = document.getElementById('levelFilter').value;
    var s = document.getElementById('searchInput').value.toLowerCase();
    markers.clearLayers();
    var html = '';
    allTrails.forEach(t => {
        if(f!=='Все' && t.level!==f) return;
        if(s && !t.name.toLowerCase().includes(s)) return;
        var c = JSON.parse(t.coordinates||'[]');
        if(c.length>0) {
            var col = {'Зелёный':'#10B981','Синий':'#3B82F6','Красный':'#EF4444'}[t.level]||'#3B82F6';
            L.polyline(c, {color:col, weight:4}).addTo(markers);
            var start = L.circleMarker(c[0], {radius:6, color:col, fillColor:'#fff', fillOpacity:1, weight:3}).addTo(markers);
            start.bindPopup('<b>'+t.name+'</b><br>'+t.level+'<br>⭐ '+t.rating);
            html += '<div class="trail-card" onclick="map.setView(['+c[0][0]+','+c[0][1]+'],14); showInfo('+JSON.stringify(t).replace(/"/g,'&quot;')+');">'+
                '<h4>'+t.name+'</h4><p>'+t.level+' | ⭐ '+t.rating+'</p>'+
                '<div style="font-size:12px;margin-top:4px;">'+renderStars(t.rating)+'</div>'+
            '</div>';
        }
    });
    document.getElementById('trailsList').innerHTML = html || '<div style="color:#888;text-align:center;padding:20px;">Трейлов не найдено</div>';
}

function searchPlace() {
    var q = document.getElementById('placeSearch').value;
    if(!q) return;
    fetch('https://nominatim.openstreetmap.org/search?format=json&q='+encodeURIComponent(q))
    .then(r=>r.json()).then(d=>{ if(d.length) map.setView([d[0].lat,d[0].lon],14); });
}

function calcElev() {
    var d = document.getElementById('elevDist').value;
    var g = document.getElementById('elevGrade').value;
    if(d && g) document.getElementById('elevResult').textContent = '⛰️ Набор высоты: '+(d*1000*g/100).toFixed(0)+' м';
}

function calcCal() {
    var w = document.getElementById('calcWeight').value;
    var d = document.getElementById('calcDist').value;
    var h = document.getElementById('calcHours').value;
    if(w && d && h) document.getElementById('calcResult').textContent = '🔥 Сожжено: '+(8.5*w*h).toFixed(0)+' ккал';
}

function toggleAddMode() {
    addMode = !addMode;
    document.getElementById('addInfo').style.display = addMode ? 'block' : 'none';
    if(!addMode) clearTemp();
}

function toggleEditMode() {
    editMode = !editMode;
    if (editMode) {
        alert('🖊️ Кликни на трейл на карте, чтобы начать редактирование');
        document.getElementById('editModeBtn').textContent = '❌ Выйти';
        document.getElementById('editModeBtn').style.background = '#EF4444';
    } else {
        document.getElementById('editModeBtn').textContent = '✏️ Правка';
        document.getElementById('editModeBtn').style.background = '#3B82F6';
        clearEditMarkers();
        selectedTrailId = null;
    }
}

function clearEditMarkers() {
    editMarkers.forEach(m => map.removeLayer(m));
    editMarkers = [];
    if (editPolyline) {
        map.removeLayer(editPolyline);
        editPolyline = null;
    }
}

function startEditTrail(trail) {
    selectedTrailId = trail.id;
    clearEditMarkers();
    var coords = JSON.parse(trail.coordinates || '[]');
    if (coords.length === 0) return;
    coords.forEach((c, index) => {
        var marker = L.marker([c[0], c[1]], {
            draggable: true,
            icon: L.divIcon({
                className: 'edit-marker',
                html: '<div style="background:#10B981;width:16px;height:16px;border-radius:50%;border:3px solid #fff;box-shadow:0 2px 10px rgba(0,0,0,0.3);cursor:grab;"></div>',
                iconSize: [16, 16]
            })
        }).addTo(map);
        marker.on('drag', function(e) {
            var latlng = e.target.getLatLng();
            coords[index] = [latlng.lat, latlng.lng];
            updateEditPolyline(coords);
        });
        marker.on('dragend', function() {
            saveEditTrail(coords);
        });
        marker.on('click', function() {
            if (coords.length <= 2) {
                alert('❌ Нельзя удалить последние 2 точки!');
                return;
            }
            if (confirm('🗑️ Удалить эту точку?')) {
                coords.splice(index, 1);
                map.removeLayer(marker);
                editMarkers = editMarkers.filter(m => m !== marker);
                updateEditPolyline(coords);
                saveEditTrail(coords);
            }
        });
        editMarkers.push(marker);
    });
    updateEditPolyline(coords);
}

function updateEditPolyline(coords) {
    if (editPolyline) {
        map.removeLayer(editPolyline);
    }
    if (coords.length > 1) {
        editPolyline = L.polyline(coords, {
            color: '#10B981',
            weight: 4,
            opacity: 0.8,
            dashArray: '5,10'
        }).addTo(map);
    }
}

function saveEditTrail(coords) {
    if (!selectedTrailId) return;
    var trail = allTrails.find(t => t.id === selectedTrailId);
    if (!trail) return;
    fetch('/api/trails/update', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            id: selectedTrailId,
            name: trail.name,
            level: trail.level,
            description: trail.description || '',
            coordinates: JSON.stringify(coords)
        })
    });
}

map.on('click', function(e) {
    if (editMode && !selectedTrailId) {
        var clicked = false;
        allTrails.forEach(t => {
            var coords = JSON.parse(t.coordinates || '[]');
            if (coords.length > 0) {
                for (var i = 0; i < coords.length; i++) {
                    var dist = map.distance(e.latlng, L.latLng(coords[i][0], coords[i][1]));
                    if (dist < 50) {
                        clicked = true;
                        startEditTrail(t);
                        break;
                    }
                }
            }
        });
        if (!clicked) {
            alert('❌ Кликни на трейл на карте (на любую его точку)');
        }
    }
});

function clearTemp() {
    tempMarkers.forEach(m=>markers.removeLayer(m));
    tempMarkers=[]; if(tempLine){markers.removeLayer(tempLine); tempLine=null;}
    newCoords=[]; // Исправлено: newCoords, а не coords
}

function finishTrail() {
    if(newCoords.length<2){alert('Добавьте хотя бы 2 точки'); return;}
    var name = prompt('Название трейла:'); if(!name) return;
    var level = prompt('Уровень (Зелёный, Синий, Красный, Чёрный):','Синий'); if(!level) return;
    var desc = prompt('Описание:','')||'';
    fetch('/api/trails/add', {
        method:'POST', headers:{'Content-Type':'application/json'},
        body:JSON.stringify({name:name, city:'Челябинск', level:level, description:desc, coordinates:JSON.stringify(newCoords)})
    }).then(r=>r.json()).then(d=>{
        if(d.success){alert('Трейл добавлен!'); clearTemp(); toggleAddMode(); loadTrails();}
        else alert('Ошибка: '+d.error);
    });
}

map.on('click', function(e){
    if(!addMode) return;
    newCoords.push([e.latlng.lat, e.latlng.lng]);
    var m = L.circleMarker([e.latlng.lat, e.latlng.lng], {radius:5, color:'#10B981', fillColor:'#fff', fillOpacity:1, weight:2}).addTo(markers);
    tempMarkers.push(m);
    if(tempLine) markers.removeLayer(tempLine);
    if(newCoords.length>1) tempLine = L.polyline(newCoords, {color:'#10B981', weight:4, opacity:0.8}).addTo(markers);
});

function showReviews(id) {
    curTrail = id;
    document.getElementById('reviewModal').classList.add('show');
    fetch('/api/trails/'+id+'/reviews').then(r=>r.json()).then(d=>{
        var html = '';
        if(d.reviews && d.reviews.length) {
            d.reviews.forEach(r=>{
                html += '<div class="review-item"><div class="review-stars">'+renderStars(r.rating)+'</div>'+
                    '<span class="review-name">'+r.user_name+'</span>'+
                    '<div>'+r.comment+'</div></div>';
            });
        } else html = '<p style="color:#9CA3AF;text-align:center;padding:20px;">Пока нет отзывов</p>';
        document.getElementById('reviewsContent').innerHTML = html;
    });
}

function closeModal() { document.getElementById('reviewModal').classList.remove('show'); }

function selStar(n) {
    selRating = n;
    document.querySelectorAll('#starSelector .star').forEach((el,i)=>{
        el.textContent = i<n ? '★' : '☆';
        el.className = i<n ? 'star active' : 'star';
    });
}

function sendReview() {
    var name = document.getElementById('reviewName').value||'Аноним';
    var comment = document.getElementById('reviewComment').value;
    if(!selRating){alert('Поставь оценку!'); return;}
    fetch('/api/reviews/add', {
        method:'POST', headers:{'Content-Type':'application/json'},
        body:JSON.stringify({trail_id:curTrail, user_name:name, rating:selRating, comment:comment})
    }).then(()=>{
        alert('Спасибо за отзыв!');
        closeModal();
        document.getElementById('reviewName').value='';
        document.getElementById('reviewComment').value='';
        selRating=0;
        document.querySelectorAll('#starSelector .star').forEach(el=>{el.textContent='☆';el.className='star';});
        loadTrails();
    });
}

var views = localStorage.getItem('velobit_views')||0;
if(!sessionStorage.getItem('visited')){views++; localStorage.setItem('velobit_views',views); sessionStorage.setItem('visited','1');}
document.getElementById('viewCount').textContent = views;

document.getElementById('searchInput').addEventListener('input', renderTrails);
document.getElementById('levelFilter').addEventListener('change', renderTrails);
loadTrails();
</script>

<!-- МАЛЕНЬКАЯ ЛЕНТА ВНИЗУ -->
<div style="position:fixed;bottom:10px;left:50%;transform:translateX(-50%);z-index:9999;background:rgba(0,0,0,0.6);backdrop-filter:blur(10px);border-radius:30px;padding:6px 14px;display:flex;gap:6px;border:1px solid rgba(255,255,255,0.08);box-shadow:0 4px 20px rgba(0,0,0,0.3);">
    <a href="/" style="color:#9CA3AF;text-decoration:none;font-size:11px;padding:4px 10px;border-radius:20px;transition:0.3s;">🗺️ Карта</a>
    <a href="/main" style="color:#10B981;text-decoration:none;font-size:11px;padding:4px 10px;border-radius:20px;transition:0.3s;background:rgba(16,185,129,0.15);">🏠 Главная</a>
    <a href="/admin/crm?password=velobit2026" style="color:#9CA3AF;text-decoration:none;font-size:11px;padding:4px 10px;border-radius:20px;transition:0.3s;">🏪 CRM</a>
    <span style="color:#333;padding:0 4px;">|</span>
    <button onclick="document.getElementById('reviewsModal').classList.add('show')" style="background:none;border:none;color:#9CA3AF;font-size:11px;padding:4px 10px;border-radius:20px;cursor:pointer;transition:0.3s;">📝 Отзывы</button>
</div>

</body>
</html>

@app.route('/')
def home():
    return render_template_string(HTML)

@app.route('/api/trails')
def get_trails():
    with sqlite3.connect(DATABASE) as conn:
        conn.row_factory = sqlite3.Row
        return jsonify([dict(r) for r in conn.execute("SELECT * FROM trails").fetchall()])

@app.route('/api/trails/stats')
def get_trails_stats():
    with sqlite3.connect(DATABASE) as conn:
        conn.row_factory = sqlite3.Row
        trails = []
        for row in conn.execute("SELECT * FROM trails").fetchall():
            t = dict(row)
            coords = json.loads(t['coordinates'] or '[]')
            dist = 0
            for i in range(len(coords)-1):
                lat1, lon1 = math.radians(coords[i][0]), math.radians(coords[i][1])
                lat2, lon2 = math.radians(coords[i+1][0]), math.radians(coords[i+1][1])
                dlat, dlon = lat2-lat1, lon2-lon1
                a = math.sin(dlat/2)**2 + math.cos(lat1)*math.cos(lat2)*math.sin(dlon/2)**2
                dist += 6371*2*math.atan2(math.sqrt(a), math.sqrt(1-a))
            t['distance_km'] = round(dist, 2)
            t['time_min'] = round((dist/15)*60)
            trails.append(t)
        return jsonify(trails)

@app.route('/api/trails/add', methods=['POST'])
def add_trail():
    data = request.get_json()
    try:
        with sqlite3.connect(DATABASE) as conn:
            conn.execute("INSERT INTO trails (name, city, level, description, coordinates) VALUES (?, ?, ?, ?, ?)",
                (data['name'], data.get('city','Челябинск'), data['level'], data.get('description',''), data['coordinates']))
        return jsonify({'success': True})
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/api/trails/<int:trail_id>/reviews')
def get_reviews(trail_id):
    with sqlite3.connect(DATABASE) as conn:
        conn.row_factory = sqlite3.Row
        reviews = conn.execute("SELECT * FROM reviews WHERE trail_id=? ORDER BY created_at DESC", (trail_id,)).fetchall()
        avg = conn.execute("SELECT AVG(rating) FROM reviews WHERE trail_id=?", (trail_id,)).fetchone()[0]
        return jsonify({'reviews': [dict(r) for r in reviews], 'avg_rating': round(avg or 0, 1)})

@app.route('/api/reviews/add', methods=['POST'])
def add_review():
    data = request.get_json()
    with sqlite3.connect(DATABASE) as conn:
        conn.execute("INSERT INTO reviews (trail_id, user_name, rating, comment) VALUES (?, ?, ?, ?)",
            (data['trail_id'], data['user_name'], data['rating'], data.get('comment','')))
        avg = conn.execute("SELECT AVG(rating) FROM reviews WHERE trail_id=?", (data['trail_id'],)).fetchone()[0]
        conn.execute("UPDATE trails SET rating=? WHERE id=?", (round(avg or 0, 1), data['trail_id']))
    return jsonify({'success': True})

@app.route('/admin/crm')
def crm():
    if request.args.get('password') != ADMIN_PASSWORD:
        return '<h1 style="color:red;text-align:center;margin-top:50px;">⛔ Доступ запрещён</h1>', 403
    with sqlite3.connect(DATABASE) as conn:
        conn.row_factory = sqlite3.Row
        partners = conn.execute("SELECT * FROM partners").fetchall()
    rows = ''.join(f'<tr><td>{p["name"]}</td><td>{p["contact_person"]}</td><td>{p["phone"]}</td><td>{p["status"]}</td><td>{p["price"]} ₽</td></tr>' for p in partners)
    return f'''
    <!DOCTYPE html>
    <html><head><meta charset="UTF-8"><title>CRM</title>
    <style>body{{background:#f4f4f9;font-family:sans-serif;padding:30px;}}
    h1{{color:#10B981;}}
    table{{width:100%;background:#fff;border-radius:12px;overflow:hidden;box-shadow:0 2px 10px rgba(0,0,0,0.05);}}
    th{{background:#1F2937;color:#fff;padding:12px;text-align:left;}}
    td{{padding:10px;border-bottom:1px solid #eee;}}
    .back{{display:inline-block;margin-bottom:16px;padding:8px 16px;background:#1F2937;color:#fff;text-decoration:none;border-radius:6px;}}
    </style></head>
    <body><a href="/" class="back">← На сайт</a><h1>🏪 Velobit CRM</h1>
    <table><thead><tr><th>Магазин</th><th>Контакт</th><th>Телефон</th><th>Статус</th><th>Цена</th></tr></thead>
    <tbody>{rows}</tbody></table></body></html>
    '''
@app.route('/main')
def main_page():
    return render_template_string('''
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>🚲 Velobit — Главная</title>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;800;900&display=swap" rel="stylesheet">
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            background: #0B0E14;
            color: #fff;
            font-family: 'Inter', sans-serif;
            min-height: 100vh;
        }
        .container { max-width: 1200px; margin: 0 auto; padding: 30px 20px; }
        
        /* ===== ШАПКА ===== */
        .header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 40px;
            flex-wrap: wrap;
            gap: 16px;
        }
        .logo { font-size: 32px; font-weight: 900; color: #10B981; }
        .nav-buttons { display: flex; gap: 12px; flex-wrap: wrap; }
        .nav-btn {
            padding: 10px 24px;
            border-radius: 12px;
            background: rgba(255,255,255,0.05);
            border: 1px solid rgba(255,255,255,0.08);
            color: #fff;
            text-decoration: none;
            font-weight: 600;
            transition: 0.3s;
        }
        .nav-btn:hover {
            background: #10B981;
            border-color: #10B981;
            transform: scale(1.02);
        }
        .nav-btn.active {
            background: #10B981;
            border-color: #10B981;
        }
        
        /* ===== ПЕРЕКЛЮЧАТЕЛЬ ===== */
        .tabs {
            display: flex;
            gap: 8px;
            margin-bottom: 30px;
            background: rgba(255,255,255,0.03);
            border-radius: 16px;
            padding: 6px;
            border: 1px solid rgba(255,255,255,0.05);
            flex-wrap: wrap;
        }
        .tab {
            padding: 12px 28px;
            border-radius: 12px;
            cursor: pointer;
            font-weight: 600;
            font-size: 15px;
            transition: 0.3s;
            color: #9CA3AF;
            border: none;
            background: none;
        }
        .tab:hover { color: #fff; }
        .tab.active {
            background: #10B981;
            color: #fff;
            box-shadow: 0 4px 20px rgba(16,185,129,0.3);
        }
        
        /* ===== КОНТЕНТ ===== */
        .content {
            background: rgba(255,255,255,0.03);
            border-radius: 20px;
            padding: 30px;
            border: 1px solid rgba(255,255,255,0.05);
            min-height: 300px;
            animation: fadeIn 0.4s ease;
        }
        .content h2 { font-size: 28px; margin-bottom: 16px; color: #10B981; }
        .content p { color: #9CA3AF; font-size: 16px; line-height: 1.7; }
        .content .grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 16px;
            margin-top: 20px;
        }
        .content .card {
            background: rgba(255,255,255,0.03);
            border-radius: 12px;
            padding: 20px;
            border: 1px solid rgba(255,255,255,0.05);
            text-align: center;
        }
        .content .card .icon { font-size: 40px; margin-bottom: 8px; }
        .content .card h4 { font-size: 16px; margin-bottom: 4px; }
        .content .card p { font-size: 13px; color: #6B7280; }
        
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(20px); }
            to { opacity: 1; transform: translateY(0); }
        }
        
        @media (max-width: 600px) {
            .header { flex-direction: column; align-items: stretch; }
            .tabs { flex-direction: column; }
            .tab { text-align: center; }
        }
    </style>
</head>
<body>
<div class="container">
    <!-- ШАПКА -->
    <div class="header">
        <div class="logo">🚲 VELOBIT</div>
        <div class="nav-buttons">
            <a href="/" class="nav-btn">🗺️ Карта</a>
            <a href="/main" class="nav-btn active">🏠 Главная</a>
            <a href="/admin/crm?password=velobit2026" class="nav-btn">🏪 CRM</a>
        </div>
    </div>

    <!-- ПЕРЕКЛЮЧАТЕЛЬ -->
    <div class="tabs" id="tabs">
        <button class="tab active" data-tab="treils">🌲 Трейлы</button>
        <button class="tab" data-tab="agents">🤖 Агенты</button>
        <button class="tab" data-tab="analytics">📊 Аналитика</button>
        <button class="tab" data-tab="team">👥 Команда</button>
    </div>

        <!-- КОНТЕНТ -->
    <div class="content" id="content">
        <div id="tabContent">
            <h2>🌲 Трейлы Челябинска</h2>
            <p>Лучшие велосипедные маршруты города. Выбирай и катайся!</p>
            <div class="grid">
                <div class="card"><div class="icon">🌲</div><h4>Городской Бор</h4><p>12.5 км · Синий</p></div>
                <div class="card"><div class="icon">⛰️</div><h4>Шершнёвский карьер</h4><p>8.2 км · Красный</p></div>
                <div class="card"><div class="icon">🌳</div><h4>Тополиная аллея</h4><p>5.4 км · Зелёный</p></div>
                <div class="card"><div class="icon">🏞️</div><h4>Каштакский бор</h4><p>15.0 км · Синий</p></div>
            </div>
        </div>
    </div>

    <!-- ===== ТАБЛИЦА ЛИДЕРОВ ===== -->
    <div style="margin-top:40px;">
        <h2 style="font-size:24px;color:#F59E0B;margin-bottom:16px;">🏆 Таблица лидеров</h2>
        <table style="width:100%;border-collapse:collapse;background:rgba(255,255,255,0.03);border-radius:12px;overflow:hidden;">
            <thead>
                <tr style="background:rgba(16,185,129,0.1);">
                    <th style="padding:12px;text-align:left;color:#F59E0B;">#</th>
                    <th style="padding:12px;text-align:left;color:#F59E0B;">Райдер</th>
                    <th style="padding:12px;text-align:center;color:#F59E0B;">Поездки</th>
                    <th style="padding:12px;text-align:center;color:#F59E0B;">Км</th>
                    <th style="padding:12px;text-align:center;color:#F59E0B;">Действия</th>
                </tr>
            </thead>
            <tbody id="leaderboardBody">
                <tr><td colspan="5" style="text-align:center;padding:20px;color:#9CA3AF;">Загрузка...</td></tr>
            </tbody>
        </table>
    </div>
    
        
<script>
 <script>
var map = L.map('map').setView([55.1644, 61.4368], 12);
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {attribution:'OSM'}).addTo(map);
var allTrails = [], markers = L.layerGroup().addTo(map);
var addMode = false, newCoords = [], tempMarkers = [], tempLine = null;
var selRating = 0, curTrail = null;

// ===== ДОБАВЬ ЭТИ СТРОКИ =====
var editMode = false;
var selectedTrailId = null;
var editMarkers = [];
var editPolyline = null;
// ===== КОНЕЦ ДОБАВЛЕНИЯ =====
       // ===== ПЕРЕКЛЮЧЕНИЕ ВКЛАДОК =====
    const tabs = document.querySelectorAll('.tab');
    const content = document.getElementById('tabContent');
 
    const tabData = {
        'treils': {
            title: '🌲 Трейлы Челябинска',
            desc: 'Лучшие велосипедные маршруты города. Выбирай и катайся!',
            cards: [
                { icon: '🌲', title: 'Городской Бор', desc: '12.5 км · Синий' },
                { icon: '⛰️', title: 'Шершнёвский карьер', desc: '8.2 км · Красный' },
                { icon: '🌳', title: 'Тополиная аллея', desc: '5.4 км · Зелёный' },
                { icon: '🏞️', title: 'Каштакский бор', desc: '15.0 км · Синий' }
            ]
        },
        'agents': {
            title: '🤖 Агенты Velobit',
            desc: 'Наши ИИ-агенты помогают находить заказы, писать контент и зарабатывать.',
            cards: [
                { icon: '🔍', title: 'Finder Agent', desc: 'Ищет заказы на биржах' },
                { icon: '💻', title: 'CodeCraft Agent', desc: 'Пишет код на заказ' },
                { icon: '📝', title: 'ContentBot', desc: 'Генерирует контент' },
                { icon: '💬', title: 'ChatBot', desc: 'Общается с клиентами' }
            ]
        },
        'analytics': {
            title: '📊 Аналитика Velobit',
            desc: 'Данные о трейлах, пользователях и заказах в реальном времени.',
            cards: [
                { icon: '📈', title: 'Популярные трейлы', desc: 'Топ маршрутов' },
                { icon: '👥', title: 'Активные райдеры', desc: 'Статистика' },
                { icon: '💰', title: 'Доход', desc: 'Monetization' },
                { icon: '📊', title: 'Отчёты', desc: 'PDF / Excel' }
            ]
        },
        'team': {
            title: '👥 Команда Velobit',
            desc: 'Мы — команда энтузиастов, создающих лучший вело-сервис в Челябинске.',
            cards: [
                { icon: '👤', title: 'Алексей', desc: 'Основатель' },
                { icon: '👤', title: 'Олег', desc: 'Разработчик' },
                { icon: '👤', title: 'Ольга', desc: 'Дизайнер' },
                { icon: '👤', title: 'Иван', desc: 'Маркетолог' }
            ]
        }
    };

    tabs.forEach(tab => {
        tab.addEventListener('click', function() {
            tabs.forEach(t => t.classList.remove('active'));
            this.classList.add('active');
            
            const tabName = this.dataset.tab;
            const data = tabData[tabName];
            
            if (data) {
                content.innerHTML = `
                    <h2>${data.title}</h2>
                    <p>${data.desc}</p>
                    <div class="grid">
                        ${data.cards.map(c => `
                            <div class="card">
                                <div class="icon">${c.icon}</div>
                                <h4>${c.title}</h4>
                                <p>${c.desc}</p>
                            </div>
                        `).join('')}
                    </div>
                `;
            }
        });
    });
// ===== РЕЖИМ ПРАВКИ С ПЕРЕТАСКИВАНИЕМ =====
var editMode = false;
var selectedTrailId = null;
var editMarkers = [];
var editPolyline = null;

function toggleEditMode() {
    editMode = !editMode;
    if (editMode) {
        alert('🖊️ Кликни на трейл на карте, чтобы начать редактирование');
        document.getElementById('editModeBtn').textContent = '❌ Выйти';
        document.getElementById('editModeBtn').style.background = '#EF4444';
    } else {
        document.getElementById('editModeBtn').textContent = '✏️ Правка';
        document.getElementById('editModeBtn').style.background = '#3B82F6';
        clearEditMarkers();
        selectedTrailId = null;
        // Убираем обработчик кликов для добавления точек (без ошибки)
        try {
            map.off('click', addEditPoint);
        } catch(e) {}
    }
}

function clearEditMarkers() {
    editMarkers.forEach(m => map.removeLayer(m));
    editMarkers = [];
    if (editPolyline) {
        map.removeLayer(editPolyline);
        editPolyline = null;
    }
}

function startEditTrail(trail) {
    selectedTrailId = trail.id;
    clearEditMarkers();
    
    var coords = JSON.parse(trail.coordinates || '[]');
    if (coords.length === 0) return;
    
    // Отображаем точки с маркерами
    coords.forEach((c, index) => {
        var marker = L.marker([c[0], c[1]], {
            draggable: true,
            icon: L.divIcon({
                className: 'edit-marker',
                html: '<div style="background:#10B981;width:16px;height:16px;border-radius:50%;border:3px solid #fff;box-shadow:0 2px 10px rgba(0,0,0,0.3);cursor:grab;"></div>',
                iconSize: [16, 16]
            })
        }).addTo(map);
        
        marker.on('drag', function(e) {
            var latlng = e.target.getLatLng();
            coords[index] = [latlng.lat, latlng.lng];
            updateEditPolyline(coords);
        });
        
        marker.on('dragend', function() {
            saveEditTrail(coords);
        });
        
        marker.on('click', function() {
            if (coords.length <= 2) {
                alert('❌ Нельзя удалить последние 2 точки!');
                return;
            }
            if (confirm('🗑️ Удалить эту точку?')) {
                coords.splice(index, 1);
                map.removeLayer(marker);
                editMarkers = editMarkers.filter(m => m !== marker);
                updateEditPolyline(coords);
                saveEditTrail(coords);
            }
        });
        
        editMarkers.push(marker);
    });
    
    updateEditPolyline(coords);
    map.on('click', addEditPoint);
}

function updateEditPolyline(coords) {
    if (editPolyline) {
        map.removeLayer(editPolyline);
    }
    if (coords.length > 1) {
        editPolyline = L.polyline(coords, {
            color: '#10B981',
            weight: 4,
            opacity: 0.8,
            dashArray: '5,10'
        }).addTo(map);
    }
}

function addEditPoint(e) {
    if (!editMode || !selectedTrailId) return;
    
    var coords = getCurrentCoords();
    if (coords.length < 2) return;
    
    // Находим ближайший сегмент
    var minDist = Infinity;
    var insertIndex = -1;
    var newPoint = null;
    
    for (var i = 0; i < coords.length - 1; i++) {
        var a = L.latLng(coords[i][0], coords[i][1]);
        var b = L.latLng(coords[i+1][0], coords[i+1][1]);
        var dist = distanceToSegment(e.latlng, a, b);
        if (dist < minDist) {
            minDist = dist;
            insertIndex = i + 1;
            var t = projectPointOnSegment(e.latlng, a, b);
            newPoint = [t.lat, t.lng];
        }
    }
    
    if (minDist < 50 && newPoint) {
        coords.splice(insertIndex, 0, newPoint);
        // Перерисовываем
        var trail = allTrails.find(t => t.id === selectedTrailId);
        if (trail) {
            clearEditMarkers();
            // Сохраняем координаты сразу
            saveEditTrail(coords);
            // Пересоздаём маркеры
            startEditTrail(trail);
        }
    }
}

function distanceToSegment(p, a, b) {
    var x = p.lng, y = p.lat;
    var x1 = a.lng, y1 = a.lat;
    var x2 = b.lng, y2 = b.lat;
    var dx = x2 - x1, dy = y2 - y1;
    var t = ((x - x1) * dx + (y - y1) * dy) / (dx*dx + dy*dy);
    t = Math.max(0, Math.min(1, t));
    var nearX = x1 + t * dx, nearY = y1 + t * dy;
    return Math.sqrt((x - nearX)*(x - nearX) + (y - nearY)*(y - nearY)) * 111000;
}

function projectPointOnSegment(p, a, b) {
    var x = p.lng, y = p.lat;
    var x1 = a.lng, y1 = a.lat;
    var x2 = b.lng, y2 = b.lat;
    var dx = x2 - x1, dy = y2 - y1;
    var t = ((x - x1) * dx + (y - y1) * dy) / (dx*dx + dy*dy);
    t = Math.max(0, Math.min(1, t));
    return L.latLng(y1 + t * dy, x1 + t * dx);
}

function getCurrentCoords() {
    var coords = [];
    editMarkers.forEach(m => {
        var latlng = m.getLatLng();
        coords.push([latlng.lat, latlng.lng]);
    });
    return coords;
}

function saveEditTrail(coords) {
    if (!selectedTrailId) return;
    var trail = allTrails.find(t => t.id === selectedTrailId);
    if (!trail) return;
    
    fetch('/api/trails/update', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            id: selectedTrailId,
            name: trail.name,
            level: trail.level,
            description: trail.description || '',
            coordinates: JSON.stringify(coords)
        })
    })
    .then(r => r.json())
    .then(data => {
        if (data.success) {
            console.log('✅ Сохранено');
        }
    });
}

// Перехват клика для выбора трейла
map.on('click', function(e) {
    if (editMode && !selectedTrailId) {
        var clicked = false;
        allTrails.forEach(t => {
            var coords = JSON.parse(t.coordinates || '[]');
            if (coords.length > 0) {
                var dist = map.distance(e.latlng, L.latLng(coords[0][0], coords[0][1]));
                if (dist < 100) {
                    clicked = true;
                    startEditTrail(t);
                }
            }
        });
        if (!clicked) {
            alert('❌ Кликни на трейл на карте (рядом с началом)');
        }
    }
});
</script>
function editRider(id) {
        const newName = prompt('Новое имя райдера:');
        if (newName && newName.trim()) {
            fetch('/api/leaderboard/update', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ id: id, username: newName.trim() })
            }).then(() => loadLeaderboard());
        }
    }

    function deleteRider(id) {
        if (confirm('Удалить райдера из таблицы?')) {
            fetch('/api/leaderboard/delete', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ id: id })
            }).then(() => loadLeaderboard());
        }
    }

    // Загружаем таблицу при открытии страницы
    document.addEventListener('DOMContentLoaded', function() {
        loadLeaderboard();
    });
        // ===== ТАБЛИЦА ЛИДЕРОВ =====
    function loadLeaderboard() {
        fetch('/api/leaderboard')
            .then(r => r.json())
            .then(data => {
                const tbody = document.getElementById('leaderboardBody');
                if (!tbody) return;
                if (data.length === 0) {
                    tbody.innerHTML = '<tr><td colspan="5" style="text-align:center;padding:20px;color:#9CA3AF;">Пока нет райдеров</td></tr>';
                    return;
                }
</body>
</html>
    ''')
@app.route('/api/trails/update', methods=['POST'])
def update_trail():
    data = request.get_json()
    try:
        with sqlite3.connect(DATABASE) as conn:
            conn.execute(
                "UPDATE trails SET name=?, level=?, description=?, coordinates=? WHERE id=?",
                (data['name'], data['level'], data.get('description', ''), data.get('coordinates', '[]'), data['id'])
            )
        return jsonify({'success': True})
    except Exception as e:
        return jsonify({'error': str(e)}), 500
@app.route('/marketplace')
def marketplace():
    return render_template_string('''
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>🛍️ Velobit — Барахолка</title>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;800;900&display=swap" rel="stylesheet">
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            background: #0B0E14;
            color: #fff;
            font-family: 'Inter', sans-serif;
            min-height: 100vh;
        }
        .container { max-width: 1000px; margin: 0 auto; padding: 30px 20px; }
        .header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 30px;
            flex-wrap: wrap;
            gap: 16px;
        }
        .logo { font-size: 28px; font-weight: 900; color: #10B981; }
        .nav-btn {
            padding: 10px 20px;
            border-radius: 12px;
            background: rgba(255,255,255,0.05);
            border: 1px solid rgba(255,255,255,0.08);
            color: #fff;
            text-decoration: none;
            font-weight: 600;
            transition: 0.3s;
        }
        .nav-btn:hover { background: #10B981; }
        .title { font-size: 32px; font-weight: 800; margin-bottom: 8px; }
        .subtitle { color: #9CA3AF; margin-bottom: 30px; }
        .grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 16px; }
        .card {
            background: rgba(255,255,255,0.03);
            border-radius: 16px;
            padding: 20px;
            border: 1px solid rgba(255,255,255,0.05);
            transition: 0.3s;
        }
        .card:hover { background: rgba(255,255,255,0.06); transform: translateY(-2px); }
        .card .icon { font-size: 40px; margin-bottom: 8px; }
        .card h4 { font-size: 16px; margin-bottom: 4px; }
        .card p { font-size: 13px; color: #9CA3AF; }
        .card .price { color: #10B981; font-weight: 700; font-size: 18px; margin-top: 8px; }
        .btn {
            display: inline-block;
            padding: 10px 24px;
            background: #10B981;
            border: none;
            border-radius: 10px;
            color: #fff;
            font-weight: 700;
            text-decoration: none;
            transition: 0.3s;
            cursor: pointer;
        }
        .btn:hover { background: #059669; transform: scale(1.02); }
        .btn-secondary { background: rgba(255,255,255,0.05); border: 1px solid rgba(255,255,255,0.08); }
        .btn-secondary:hover { background: rgba(255,255,255,0.1); }
        .form { background: rgba(255,255,255,0.03); border-radius: 16px; padding: 20px; margin-top: 20px; border: 1px solid rgba(255,255,255,0.05); }
        .form input { width: 100%; padding: 10px; border-radius: 8px; border: 1px solid #333; background: #1a1a2e; color: #fff; margin-bottom: 8px; }
        .form textarea { width: 100%; padding: 10px; border-radius: 8px; border: 1px solid #333; background: #1a1a2e; color: #fff; resize: vertical; }
        .form button { padding: 10px 20px; background: #10B981; border: none; border-radius: 8px; color: #fff; font-weight: 700; cursor: pointer; }
        @media (max-width: 600px) {
            .header { flex-direction: column; align-items: stretch; }
            .title { font-size: 24px; }
        }
    </style>
</head>
<body>
<div class="container">
    <div class="header">
        <div class="logo">🛍️ VELOBIT</div>
        <div>
            <a href="/" class="nav-btn">🗺️ Карта</a>
            <a href="/main" class="nav-btn">🏠 Главная</a>
            <a href="/admin/crm?password=velobit2026" class="nav-btn">🏪 CRM</a>
        </div>
    </div>

    <div class="title">🛍️ Барахолка Velobit</div>
    <div class="subtitle">Продавай и покупай велосипеды, запчасти, экипировку</div>

    <div class="grid">
        <div class="card">
            <div class="icon">🚲</div>
            <h4>Велосипед горный</h4>
            <p>Отличное состояние, 2024 год</p>
            <div class="price">25 000 ₽</div>
        </div>
        <div class="card">
            <div class="icon">🪢</div>
            <h4>Шлем велосипедный</h4>
            <p>Размер M, новый</p>
            <div class="price">3 500 ₽</div>
        </div>
        <div class="card">
            <div class="icon">🔧</div>
            <h4>Набор инструментов</h4>
            <p>Для велосипеда, 15 предметов</p>
            <div class="price">2 000 ₽</div>
        </div>
    </div>

    <div class="form">
        <h3 style="margin-bottom:12px;color:#F59E0B;">📝 Добавить объявление</h3>
        <input type="text" id="adTitle" placeholder="Название товара">
        <textarea id="adDesc" rows="3" placeholder="Описание"></textarea>
        <input type="number" id="adPrice" placeholder="Цена (₽)">
        <button onclick="addAd()">✅ Добавить объявление</button>
        <div id="adResult" style="margin-top:8px;color:#10B981;"></div>
    </div>
</div>

<script>
function addAd() {
    var title = document.getElementById('adTitle').value;
    var desc = document.getElementById('adDesc').value;
    var price = document.getElementById('adPrice').value;
    if (!title || !price) { alert('Заполните все поля'); return; }
    document.getElementById('adResult').textContent = '✅ Объявление добавлено!';
    document.getElementById('adTitle').value = '';
    document.getElementById('adDesc').value = '';
    document.getElementById('adPrice').value = '';
}
</script>
</body>
</html>
"""
@app.route('/api/leaderboard')
def get_leaderboard():
    with sqlite3.connect(DATABASE) as conn:
        conn.row_factory = sqlite3.Row
        users = conn.execute("SELECT * FROM users ORDER BY total_km DESC LIMIT 10").fetchall()
        return jsonify([dict(u) for u in users])





@app.route('/api/leaderboard/update', methods=['POST'])
def update_leaderboard():
    data = request.get_json()
    with sqlite3.connect(DATABASE) as conn:
        conn.execute("UPDATE users SET username=? WHERE id=?", (data['username'], data['id']))
    return jsonify({'success': True})

@app.route('/api/leaderboard/delete', methods=['POST'])
def delete_leaderboard():
    data = request.get_json()
    with sqlite3.connect(DATABASE) as conn:
        conn.execute("DELETE FROM users WHERE id=?", (data['id'],))
    return jsonify({'success': True})
if __name__ == '__main__':
    app.run(debug=True, port=5000)
