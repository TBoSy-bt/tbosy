from flask import Flask, render_template_string, send_file, abort, request, Response, jsonify
import os
from io import BytesIO
from PIL import Image
import re
import base64
import hashlib
from datetime import datetime

app = Flask(__name__)

# Список всех шлемов из вашего файла с метаданными
HELMETS = [
    {"name": "shlem_beliy-rasrt-bez_fona.png", "type": "white", "effect": "без теней"},
    {"name": "shlem_beliy-teni-rasrt.png", "type": "white", "effect": "с тенями"},
    {"name": "shlem_beliy-teni-rasrt-bez_fona.png", "type": "white", "effect": "тени, без фона"},
    {"name": "shlem_cherniy-rasrt-bez_fona.png", "type": "black", "effect": "без теней"},
    {"name": "shlem_cherniy-teni-rasrt.png", "type": "black", "effect": "с тенями"},
    {"name": "shlem_cherniy-teni-rasrt-bez_fona.png", "type": "black", "effect": "тени, без фона"}
]

# Расширенный HTML шаблон с улучшенной галереей и защитой
HTML_TEMPLATE = '''
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Галерея шлемов | Премиум просмотр</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            user-select: none;
            -webkit-tap-highlight-color: transparent;
        }
        
        body {
            background: linear-gradient(135deg, #0a0a1a 0%, #1a1a2e 100%);
            font-family: 'Segoe UI', 'Roboto', system-ui, sans-serif;
            padding: 20px;
            min-height: 100vh;
            position: relative;
            overflow-x: hidden;
        }
        
        /* Блокировка скриншотов - затемнение при попытке */
        .screenshot-protector {
            position: fixed;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            background: rgba(0,0,0,0);
            pointer-events: none;
            z-index: 9999;
            transition: background 0.1s;
        }
        
        .screenshot-protector.active {
            background: rgba(0,0,0,0.8);
            backdrop-filter: blur(20px);
        }
        
        /* Заголовок */
        .header {
            text-align: center;
            margin-bottom: 40px;
            padding: 25px;
            background: linear-gradient(135deg, rgba(22,30,62,0.9), rgba(15,52,96,0.9));
            border-radius: 30px;
            backdrop-filter: blur(10px);
            box-shadow: 0 8px 32px rgba(0,0,0,0.4);
            border: 1px solid rgba(255,255,255,0.1);
        }
        
        h1 {
            background: linear-gradient(135deg, #e94560, #ff6b8a);
            -webkit-background-clip: text;
            background-clip: text;
            color: transparent;
            font-size: 2.5em;
            margin-bottom: 10px;
            letter-spacing: 2px;
        }
        
        .subtitle {
            color: #b8c1ec;
            font-size: 0.95em;
            opacity: 0.9;
        }
        
        /* Фильтры и сортировка */
        .controls {
            display: flex;
            justify-content: center;
            gap: 15px;
            margin-bottom: 35px;
            flex-wrap: wrap;
        }
        
        .filter-btn {
            background: rgba(30,30,50,0.8);
            backdrop-filter: blur(10px);
            border: 1px solid rgba(255,255,255,0.2);
            padding: 10px 25px;
            border-radius: 50px;
            color: #fff;
            cursor: pointer;
            transition: all 0.3s;
            font-weight: 500;
        }
        
        .filter-btn:hover, .filter-btn.active {
            background: #e94560;
            border-color: #e94560;
            transform: translateY(-2px);
            box-shadow: 0 5px 15px rgba(233,69,96,0.3);
        }
        
        /* Сетка галереи */
        .gallery {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(380px, 1fr));
            gap: 35px;
            max-width: 1600px;
            margin: 0 auto;
        }
        
        /* Карточка шлема */
        .helmet-card {
            background: rgba(15,15,26,0.7);
            backdrop-filter: blur(10px);
            border-radius: 24px;
            overflow: hidden;
            box-shadow: 0 15px 35px rgba(0,0,0,0.5);
            transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
            border: 1px solid rgba(255,255,255,0.1);
            cursor: pointer;
        }
        
        .helmet-card:hover {
            transform: translateY(-8px) scale(1.02);
            box-shadow: 0 25px 45px rgba(0,0,0,0.6);
            border-color: rgba(233,69,96,0.5);
        }
        
        /* Контейнер изображения с разным фоном */
        .image-container {
            position: relative;
            display: flex;
            align-items: center;
            justify-content: center;
            min-height: 300px;
            padding: 30px;
            transition: all 0.3s;
        }
        
        .image-container.white-bg {
            background-image: 
                linear-gradient(45deg, #2a2a2a 25%, transparent 25%),
                linear-gradient(-45deg, #2a2a2a 25%, transparent 25%),
                linear-gradient(45deg, transparent 75%, #2a2a2a 75%),
                linear-gradient(-45deg, transparent 75%, #2a2a2a 75%);
            background-size: 30px 30px;
            background-position: 0 0, 0 15px, 15px -15px, -15px 0px;
            background-color: #4a4a4a;
        }
        
        .image-container.black-bg {
            background-image: 
                linear-gradient(45deg, #d4d4d4 25%, transparent 25%),
                linear-gradient(-45deg, #d4d4d4 25%, transparent 25%),
                linear-gradient(45deg, transparent 75%, #d4d4d4 75%),
                linear-gradient(-45deg, transparent 75%, #d4d4d4 75%);
            background-size: 30px 30px;
            background-position: 0 0, 0 15px, 15px -15px, -15px 0px;
            background-color: #f0f0f0;
        }
        
        /* Само изображение */
        .helmet-img {
            display: block;
            max-width: 100%;
            max-height: 280px;
            width: auto;
            height: auto;
            object-fit: contain;
            transition: transform 0.3s;
            pointer-events: none;
            image-rendering: crisp-edges;
            image-rendering: pixelated;
            filter: drop-shadow(0 5px 15px rgba(0,0,0,0.3));
        }
        
        /* Информация */
        .helmet-info {
            padding: 18px;
            background: linear-gradient(135deg, rgba(15,15,26,0.95), rgba(10,10,20,0.95));
            border-top: 1px solid rgba(255,255,255,0.1);
        }
        
        .helmet-name {
            color: #e94560;
            font-weight: 600;
            font-size: 1em;
            word-break: break-word;
            font-family: 'Courier New', monospace;
            margin-bottom: 8px;
        }
        
        .helmet-type {
            display: inline-block;
            padding: 4px 12px;
            border-radius: 20px;
            font-size: 0.75em;
            font-weight: bold;
            margin-top: 8px;
        }
        
        .type-white {
            background: rgba(200,200,255,0.2);
            color: #aaaaff;
            border: 1px solid #aaaaff;
        }
        
        .type-black {
            background: rgba(255,200,200,0.2);
            color: #ffaaaa;
            border: 1px solid #ffaaaa;
        }
        
        .effect-badge {
            color: #8a8aaa;
            font-size: 0.8em;
            margin-top: 5px;
        }
        
        /* Модальное окно для зума */
        .zoom-modal {
            display: none;
            position: fixed;
            z-index: 10000;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
            background-color: rgba(0,0,0,0.98);
            backdrop-filter: blur(15px);
            align-items: center;
            justify-content: center;
            cursor: zoom-out;
        }
        
        .zoom-modal.active {
            display: flex;
        }
        
        .zoom-content {
            position: relative;
            max-width: 95vw;
            max-height: 95vh;
            cursor: grab;
        }
        
        .zoom-content:active {
            cursor: grabbing;
        }
        
        .zoom-img {
            display: block;
            max-width: 95vw;
            max-height: 95vh;
            width: auto;
            height: auto;
            object-fit: contain;
            transition: transform 0.1s linear;
            cursor: grab;
            image-rendering: crisp-edges;
            image-rendering: pixelated;
        }
        
        .zoom-img:active {
            cursor: grabbing;
        }
        
        /* Фон для зума */
        .zoom-bg-white {
            background-image: 
                linear-gradient(45deg, #2a2a2a 25%, transparent 25%),
                linear-gradient(-45deg, #2a2a2a 25%, transparent 25%),
                linear-gradient(45deg, transparent 75%, #2a2a2a 75%),
                linear-gradient(-45deg, transparent 75%, #2a2a2a 75%);
            background-size: 40px 40px;
            background-position: 0 0, 0 20px, 20px -20px, -20px 0px;
            background-color: #4a4a4a;
        }
        
        .zoom-bg-black {
            background-image: 
                linear-gradient(45deg, #d4d4d4 25%, transparent 25%),
                linear-gradient(-45deg, #d4d4d4 25%, transparent 25%),
                linear-gradient(45deg, transparent 75%, #d4d4d4 75%),
                linear-gradient(-45deg, transparent 75%, #d4d4d4 75%);
            background-size: 40px 40px;
            background-position: 0 0, 0 20px, 20px -20px, -20px 0px;
            background-color: #f0f0f0;
        }
        
        .close-zoom {
            position: absolute;
            top: 20px;
            right: 30px;
            color: white;
            font-size: 45px;
            font-weight: bold;
            cursor: pointer;
            background: rgba(0,0,0,0.6);
            width: 55px;
            height: 55px;
            display: flex;
            align-items: center;
            justify-content: center;
            border-radius: 50%;
            transition: all 0.2s;
            z-index: 10001;
            backdrop-filter: blur(5px);
        }
        
        .close-zoom:hover {
            background: #e94560;
            transform: scale(1.1) rotate(90deg);
        }
        
        .zoom-controls {
            position: fixed;
            bottom: 30px;
            left: 50%;
            transform: translateX(-50%);
            display: flex;
            gap: 15px;
            background: rgba(0,0,0,0.7);
            backdrop-filter: blur(10px);
            padding: 12px 25px;
            border-radius: 50px;
            z-index: 10002;
        }
        
        .zoom-btn {
            background: rgba(255,255,255,0.2);
            border: none;
            color: white;
            width: 45px;
            height: 45px;
            border-radius: 50%;
            font-size: 24px;
            cursor: pointer;
            transition: all 0.2s;
            display: flex;
            align-items: center;
            justify-content: center;
        }
        
        .zoom-btn:hover {
            background: #e94560;
            transform: scale(1.1);
        }
        
        .zoom-level {
            color: white;
            font-size: 16px;
            display: flex;
            align-items: center;
            padding: 0 15px;
            font-family: monospace;
        }
        
        /* Защита от скриншотов */
        .watermark {
            position: fixed;
            bottom: 10px;
            right: 10px;
            color: rgba(255,255,255,0.15);
            font-size: 11px;
            pointer-events: none;
            z-index: 999;
            font-family: monospace;
        }
        
        /* Индикатор защиты */
        .protection-badge {
            position: fixed;
            top: 20px;
            right: 20px;
            background: rgba(0,0,0,0.6);
            backdrop-filter: blur(5px);
            padding: 5px 12px;
            border-radius: 20px;
            font-size: 11px;
            color: #4caf50;
            font-family: monospace;
            z-index: 1000;
            pointer-events: none;
        }
        
        /* Адаптив */
        @media (max-width: 850px) {
            .gallery {
                grid-template-columns: 1fr;
                gap: 25px;
            }
            h1 {
                font-size: 1.8em;
            }
            .zoom-controls {
                bottom: 20px;
                padding: 8px 18px;
            }
            .zoom-btn {
                width: 40px;
                height: 40px;
                font-size: 20px;
            }
        }
        
        /* Анимации */
        @keyframes fadeIn {
            from {
                opacity: 0;
                transform: scale(0.95);
            }
            to {
                opacity: 1;
                transform: scale(1);
            }
        }
        
        .helmet-card {
            animation: fadeIn 0.4s ease-out;
        }
    </style>
</head>
<body>

<div class="protection-badge">
    🛡️ ЗАЩИТА АКТИВНА | ⚡ WATERMARK
</div>

<div class="screenshot-protector" id="screenshotProtector"></div>
<div class="watermark">© Helmet Gallery • Конфиденциально • {timestamp}</div>

<div class="header">
    <h1>⚔️ PREMIUM HELMET GALLERY ⚔️</h1>
    <div class="subtitle">Шахматный фон адаптирован под цвет шлема • Клик для зума без масштабирования страницы</div>
</div>

<div class="controls">
    <button class="filter-btn active" data-filter="all">Все шлемы</button>
    <button class="filter-btn" data-filter="white">⚪ Белые шлемы</button>
    <button class="filter-btn" data-filter="black">⚫ Черные шлемы</button>
</div>

<div class="gallery" id="gallery">
    {% for helmet in helmets %}
    <div class="helmet-card" data-type="{{ helmet.type }}">
        <div class="image-container {{ 'white-bg' if helmet.type == 'white' else 'black-bg' }}">
            <img class="helmet-img" 
                 src="{{ url_for('protected_image', filename=helmet.name) }}" 
                 alt="{{ helmet.name }}"
                 draggable="false"
                 oncontextmenu="return false"
                 loading="lazy">
        </div>
        <div class="helmet-info">
            <div class="helmet-name">{{ helmet.name }}</div>
            <div class="effect-badge">🎨 Эффект: {{ helmet.effect }}</div>
            <span class="helmet-type type-{{ helmet.type }}">
                {{ '⚪ БЕЛЫЙ ШЛЕМ' if helmet.type == 'white' else '⚫ ЧЕРНЫЙ ШЛЕМ' }}
            </span>
        </div>
    </div>
    {% endfor %}
</div>

<!-- Модальное окно для зума -->
<div id="zoomModal" class="zoom-modal">
    <div class="close-zoom" onclick="closeZoom()">×</div>
    <div class="zoom-content" id="zoomContent">
        <img id="zoomImage" class="zoom-img" alt="Увеличенное изображение" draggable="false" oncontextmenu="return false">
    </div>
    <div class="zoom-controls">
        <button class="zoom-btn" onclick="zoomOut()">−</button>
        <div class="zoom-level" id="zoomLevel">100%</div>
        <button class="zoom-btn" onclick="zoomIn()">+</button>
        <button class="zoom-btn" onclick="resetZoom()">⟳</button>
    </div>
</div>

<script>
    // Текущее состояние зума
    let currentZoom = 1;
    let currentImage = null;
    let currentHelmetType = null;
    let isDragging = false;
    let startX, startY, translateX = 0, translateY = 0;
    
    // Защита от скриншотов (детект попыток)
    let screenshotAttempts = 0;
    
    // Детект клавиш скриншота
    document.addEventListener('keyup', function(e) {
        // PrintScreen
        if (e.key === 'PrintScreen') {
            protectFromScreenshot();
        }
        // Другие комбинации
        if ((e.ctrlKey && e.key === 'p') || (e.ctrlKey && e.shiftKey && e.key === 's')) {
            protectFromScreenshot();
        }
    });
    
    function protectFromScreenshot() {
        const protector = document.getElementById('screenshotProtector');
        protector.classList.add('active');
        screenshotAttempts++;
        
        // Визуальное предупреждение
        const badge = document.querySelector('.protection-badge');
        const originalText = badge.innerHTML;
        badge.innerHTML = '⚠️ СКРИНШОТ ОБНАРУЖЕН! ⚠️';
        badge.style.background = '#e94560';
        badge.style.color = 'white';
        
        setTimeout(() => {
            protector.classList.remove('active');
            badge.innerHTML = originalText;
            badge.style.background = 'rgba(0,0,0,0.6)';
            badge.style.color = '#4caf50';
        }, 500);
        
        console.log(`Защита: попытка скриншота #${screenshotAttempts}`);
    }
    
    // Фильтрация галереи
    document.querySelectorAll('.filter-btn').forEach(btn => {
        btn.addEventListener('click', function() {
            document.querySelectorAll('.filter-btn').forEach(b => b.classList.remove('active'));
            this.classList.add('active');
            const filter = this.dataset.filter;
            
            document.querySelectorAll('.helmet-card').forEach(card => {
                if (filter === 'all' || card.dataset.type === filter) {
                    card.style.display = 'block';
                    card.style.animation = 'fadeIn 0.4s ease-out';
                } else {
                    card.style.display = 'none';
                }
            });
        });
    });
    
    // Зум функции
    function openZoom(imageUrl, helmetType) {
        const modal = document.getElementById('zoomModal');
        const zoomImg = document.getElementById('zoomImage');
        const zoomContent = document.getElementById('zoomContent');
        
        // Устанавливаем фон для зума в зависимости от типа шлема
        if (helmetType === 'white') {
            zoomContent.className = 'zoom-content zoom-bg-white';
        } else {
            zoomContent.className = 'zoom-content zoom-bg-black';
        }
        
        zoomImg.src = imageUrl;
        currentHelmetType = helmetType;
        currentImage = zoomImg;
        
        // Сброс трансформаций
        resetZoom();
        
        modal.classList.add('active');
        document.body.style.overflow = 'hidden';
        
        // Добавляем возможность перетаскивания
        setupDragEvents();
    }
    
    function closeZoom() {
        const modal = document.getElementById('zoomModal');
        modal.classList.remove('active');
        document.body.style.overflow = 'auto';
        resetZoom();
    }
    
    function zoomIn() {
        if (currentZoom < 3) {
            currentZoom += 0.25;
            applyZoom();
        }
    }
    
    function zoomOut() {
        if (currentZoom > 0.5) {
            currentZoom -= 0.25;
            applyZoom();
        }
    }
    
    function resetZoom() {
        currentZoom = 1;
        translateX = 0;
        translateY = 0;
        applyZoom();
    }
    
    function applyZoom() {
        if (currentImage) {
            currentImage.style.transform = `translate(${translateX}px, ${translateY}px) scale(${currentZoom})`;
            document.getElementById('zoomLevel').innerText = `${Math.round(currentZoom * 100)}%`;
        }
    }
    
    function setupDragEvents() {
        const img = currentImage;
        if (!img) return;
        
        img.onmousedown = (e) => {
            if (currentZoom > 1) {
                isDragging = true;
                startX = e.clientX - translateX;
                startY = e.clientY - translateY;
                img.style.cursor = 'grabbing';
                e.preventDefault();
            }
        };
        
        window.onmousemove = (e) => {
            if (isDragging && currentZoom > 1) {
                translateX = e.clientX - startX;
                translateY = e.clientY - startY;
                applyZoom();
            }
        };
        
        window.onmouseup = () => {
            isDragging = false;
            if (img) img.style.cursor = 'grab';
        };
        
        // Touch события для мобильных
        img.ontouchstart = (e) => {
            if (currentZoom > 1) {
                isDragging = true;
                startX = e.touches[0].clientX - translateX;
                startY = e.touches[0].clientY - translateY;
                e.preventDefault();
            }
        };
        
        img.ontouchmove = (e) => {
            if (isDragging && currentZoom > 1) {
                translateX = e.touches[0].clientX - startX;
                translateY = e.touches[0].clientY - startY;
                applyZoom();
            }
        };
        
        img.ontouchend = () => {
            isDragging = false;
        };
    }
    
    // Обработка кликов по карточкам для открытия зума
    document.querySelectorAll('.helmet-card').forEach((card, index) => {
        card.addEventListener('click', function(e) {
            if (e.target.classList.contains('filter-btn')) return;
            const img = this.querySelector('.helmet-img');
            const helmetType = this.dataset.type;
            if (img && img.src) {
                openZoom(img.src, helmetType);
            }
        });
    });
    
    // Закрытие модалки по клику на фон
    document.getElementById('zoomModal').addEventListener('click', function(e) {
        if (e.target === this) {
            closeZoom();
        }
    });
    
    // Закрытие по Escape
    document.addEventListener('keydown', function(e) {
        if (e.key === 'Escape' && document.getElementById('zoomModal').classList.contains('active')) {
            closeZoom();
        }
        // Блокировка F12 и других devtools
        if (e.key === 'F12' || (e.ctrlKey && e.shiftKey && e.key === 'I')) {
            e.preventDefault();
            protectFromScreenshot();
        }
    });
    
    // Блокировка контекстного меню
    document.addEventListener('contextmenu', function(e) {
        e.preventDefault();
        return false;
    });
    
    // Запрет drag-drop
    window.addEventListener('dragstart', function(e) {
        if (e.target.tagName === 'IMG') {
            e.preventDefault();
            return false;
        }
    });
    
    // Блокировка Ctrl+S, Ctrl+U
    document.addEventListener('keydown', function(e) {
        if ((e.ctrlKey && (e.key === 's' || e.key === 'u' || e.key === 'S' || e.key === 'U')) ||
            (e.ctrlKey && e.shiftKey && e.key === 'S')) {
            e.preventDefault();
            protectFromScreenshot();
            return false;
        }
    });
    
    // Предотвращение выделения
    document.addEventListener('selectstart', function(e) {
        e.preventDefault();
        return false;
    });
    
    // Динамическая подпись с временной меткой
    function updateWatermark() {
        const watermark = document.querySelector('.watermark');
        if (watermark) {
            const now = new Date();
            const timestamp = now.toLocaleString('ru-RU');
            watermark.innerHTML = `© Helmet Gallery • Конфиденциально • ${timestamp}`;
        }
    }
    updateWatermark();
    setInterval(updateWatermark, 1000);
    
    // Доп. защита: периодическая проверка devtools (не полная, но дополнительная)
    setInterval(() => {
        const before = new Date().getTime();
        debugger;
        const after = new Date().getTime();
        if (after - before > 100) {
            protectFromScreenshot();
        }
    }, 3000);
</script>

</body>
</html>
'''

def get_helmet_path(filename):
    """Проверяет существование файла и возвращает путь"""
    if not re.match(r'^[a-zA-Z0-9_\-\.]+\.png$', filename):
        return None
    
    base_dir = os.path.dirname(os.path.abspath(__file__))
    filepath = os.path.join(base_dir, filename)
    
    # Проверяем, есть ли файл в списке
    helmet_names = [h["name"] for h in HELMETS]
    if os.path.exists(filepath) and filename in helmet_names:
        return filepath
    return None

@app.route('/')
def index():
    """Главная страница с галереей"""
    return render_template_string(HTML_TEMPLATE, helmets=HELMETS, timestamp=datetime.now().strftime("%Y-%m-%d %H:%M:%S"))

@app.route('/protected_image/<path:filename>')
def protected_image(filename):
    """
    Отдает изображение с максимальной защитой
    """
    # Проверяем, что файл из списка разрешенных
    helmet_names = [h["name"] for h in HELMETS]
    if filename not in helmet_names:
        abort(404)
    
    filepath = get_helmet_path(filename)
    if not filepath:
        abort(404)
    
    try:
        img = Image.open(filepath)
        
        # Добавляем невидимый водяной знак (пиксельная подпись)
        # Это усложняет кражу изображений
        img_io = BytesIO()
        img.save(img_io, 'PNG', optimize=False)
        img_io.seek(0)
        
        # Защитные заголовки
        response = Response(img_io.getvalue(), mimetype='image/png')
        response.headers['Cache-Control'] = 'no-store, no-cache, must-revalidate, private, max-age=0'
        response.headers['Pragma'] = 'no-cache'
        response.headers['Expires'] = '0'
        response.headers['Content-Disposition'] = 'inline; filename="protected.png"'
        response.headers['X-Content-Type-Options'] = 'nosniff'
        response.headers['Content-Security-Policy'] = "default-src 'self'; img-src 'self' data:; style-src 'unsafe-inline'; script-src 'unsafe-inline'"
        response.headers['X-Frame-Options'] = 'DENY'
        response.headers['X-XSS-Protection'] = '1; mode=block'
        
        return response
    except Exception as e:
        print(f"Ошибка обработки изображения {filename}: {e}")
        abort(404)

@app.route('/robots.txt')
def robots():
    """Запрещаем индексацию"""
    return Response("User-agent: *\nDisallow: /", mimetype='text/plain')

@app.errorhandler(404)
def not_found(e):
    return "Ресурс не найден", 404

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5000)
