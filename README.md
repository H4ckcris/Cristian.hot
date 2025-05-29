[<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <title>Corazón Cósmico con Interacción Móvil</title>
  <style>
    html, body {
      margin: 0;
      padding: 0;
      overflow: hidden;
      background: black;
      font-family: sans-serif;
      color: white;
    }
    canvas {
      display: block;
      position: absolute;
      top: 0;
      left: 0;
    }
    #background {
      position: absolute;
      width: 100%;
      height: 100%;
      background: radial-gradient(circle at center, #000010 0%, #000018 40%, #00001f 100%);
      animation: skywave 10s ease-in-out infinite alternate;
      z-index: -1;
    }
    @keyframes skywave {
      0% { background-position: 50% 50%; background-size: 100% 100%; }
      100% { background-position: 48% 52%; background-size: 110% 110%; }
    }
    #permissionButton {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      padding: 15px 30px;
      font-size: 1.2em;
      background-color: #663399;
      color: white;
      border: none;
      border-radius: 8px;
      cursor: pointer;
      z-index: 1000;
      display: none; /* Se mostrará solo si es necesario */
    }
    #permissionButton:hover {
      background-color: #7a47b0;
    }
  </style>
</head>
<body>
  <div id="background"></div>
  <canvas id="canvas"></canvas>
  <button id="permissionButton">Habilitar Interacción del Dispositivo</button>

  <script>
    const canvas = document.getElementById("canvas");
    const ctx = canvas.getContext("2d");
    let w, h;

    function resize() {
      w = canvas.width = window.innerWidth;
      h = canvas.height = window.innerHeight;
    }
    window.addEventListener("resize", resize);
    resize();

    const microbots = [];
    const numBots = 8000;
    const mouse = { x: 0, y: 0, radius: 120 };

    // Variables para el acelerómetro/giroscopio
    let deviceOrientationX = 0; // beta: front to back
    let deviceOrientationY = 0; // gamma: left to right
    const maxTiltOffset = 50; // Desplazamiento máximo del efecto por inclinación

    // Función para solicitar permisos (especialmente para iOS 13+)
    const permissionButton = document.getElementById('permissionButton');
    if (typeof DeviceOrientationEvent !== 'undefined' && typeof DeviceOrientationEvent.requestPermission === 'function') {
        // En iOS 13+, mostramos el botón para solicitar permiso
        permissionButton.style.display = 'block';
        permissionButton.addEventListener('click', () => {
            DeviceOrientationEvent.requestPermission()
                .then(response => {
                    if (response === 'granted') {
                        window.addEventListener('deviceorientation', handleDeviceOrientation);
                        permissionButton.style.display = 'none'; // Ocultar el botón
                    } else {
                        console.warn('Permiso de orientación denegado.');
                        permissionButton.textContent = 'Permiso denegado.';
                        permissionButton.disabled = true;
                    }
                })
                .catch(console.error);
        });
    } else {
        // Para otros navegadores o Android, intentamos añadir el listener directamente
        window.addEventListener('deviceorientation', handleDeviceOrientation);
        permissionButton.style.display = 'none'; // Asegurarse de que el botón no se muestre
    }

    function handleDeviceOrientation(event) {
        // beta: front to back motion (X-axis)
        // gamma: left to right motion (Y-axis)
        // alpha: rotation around Z-axis (brújula, no la usaremos directamente para el desplazamiento)

        // Normalizamos los valores (beta y gamma suelen ir de -90 a 90, o -180 a 180)
        // Escalamos para un efecto más sutil y lo mapeamos a un rango de desplazamiento
        deviceOrientationX = (event.gamma / 90) * maxTiltOffset; // Mueve en X (horizontal) con inclinación lateral
        deviceOrientationY = (event.beta / 90) * maxTiltOffset;  // Mueve en Y (vertical) con inclinación frontal/trasera

        // Aseguramos que no exceda el desplazamiento máximo
        deviceOrientationX = Math.max(-maxTiltOffset, Math.min(maxTiltOffset, deviceOrientationX));
        deviceOrientationY = Math.max(-maxTiltOffset, Math.min(maxTiltOffset, deviceOrientationY));
    }


    function heartShape(t, scale = 18) {
      const x = scale * 16 * Math.pow(Math.sin(t), 3);
      const y = -scale * (13 * Math.cos(t) - 5 * Math.cos(2*t) - 2 * Math.cos(3*t) - Math.cos(4*t));
      return [x, y];
    }

    function randomHeartColor() {
      const colors = [
        "#ff4d4d", "#ff6699", "#ff0066", "#ff5e78", "#ff85b3", "#d36ba6",
        "#ff3366", "#e75480", "#db7093", "#ff1493", "#ff69b4",
        "#66ccff", "#3399ff", "#66aaff", "#88ccee", "#aaddff",
        "#77ddff", "#99ccff", "#66ffff", "#00ccff", "#33ddff",
        "#6699ff", "#ccddff", "#aaccee", "#99e6ff", "#55bbff"
      ];
      return colors[Math.floor(Math.random() * colors.length)];
    }

    const pulseAmplitude = 0.005;
    const pulseFrequency = 0.05;

    while (microbots.length < numBots) {
      const t = Math.random() * Math.PI * 2;
      const [hx, hy] = heartShape(t, 18);
      const rand = Math.pow(Math.random(), 0.7);
      const x = w/2 + hx * rand + (Math.random() - 0.5) * 2;
      const y = h/2 + hy * rand + (Math.random() - 0.5) * 2;

      microbots.push({
        x, y,
        baseX: x,
        baseY: y,
        vx: 0,
        vy: 0,
        color: randomHeartColor(),
        initialX: x,
        initialY: y
      });
    }

    window.addEventListener("mousemove", e => {
      mouse.x = e.clientX;
      mouse.y = e.clientY;
    });
    window.addEventListener("touchmove", e => {
      if (e.touches.length > 0) {
        mouse.x = e.touches[0].clientX;
        mouse.y = e.touches[0].clientY;
      }
    }, {passive: true});

    const starfield = [];
    for (let i = 0; i < 300; i++) {
      starfield.push({
        x: Math.random() * w,
        y: Math.random() * h,
        r: Math.random() * 1.5,
        alpha: Math.random(),
        speed: 0.2 + Math.random() * 0.3,
        blinkSpeed: 0.01 + Math.random() * 0.05,
        blinkPhase: Math.random() * Math.PI * 2
      });
    }

    function drawStars() {
      ctx.save();
      // Aplica el desplazamiento del sensor a la posición de las estrellas
      ctx.translate(deviceOrientationX, deviceOrientationY);
      for (let s of starfield) {
        s.alpha = 0.3 + 0.3 * Math.sin(Date.now() * s.blinkSpeed + s.blinkPhase);
        ctx.globalAlpha = s.alpha;

        ctx.beginPath();
        ctx.arc(s.x, s.y, s.r, 0, Math.PI * 2);
        ctx.fillStyle = "#ffffff";
        ctx.fill();
        s.y += s.speed;
        if (s.y > h) {
          s.y = 0;
          s.x = Math.random() * w;
        }
      }
      ctx.restore();
    }

    let meteoritos = [];
    let impactParticles = [];

    function createMeteor() {
      const edge = Math.floor(Math.random() * 4);
      let x, y;
      switch (edge) {
        case 0: x = -50; y = Math.random() * h; break;
        case 1: x = Math.random() * w; y = -50; break;
        case 2: x = w + 50; y = Math.random() * h; break;
        case 3: x = Math.random() * w; y = h + 50; break;
      }

      const dx = (w / 2) - x;
      const dy = (h / 2) - y;
      const dist = Math.sqrt(dx*dx + dy*dy);
      const speed = 10;
      const vx = (dx / dist) * speed;
      const vy = (dy / dist) * speed;

      meteoritos.push({
        x, y, vx, vy,
        trail: [],
        hasImpacted: false
      });
      
      // Añadir una vibración sutil al crear un meteorito (si la API está disponible)
      if (navigator.vibrate) {
          navigator.vibrate(50); // Vibración corta de 50ms
      }
    }

    canvas.addEventListener("dblclick", createMeteor);
    canvas.addEventListener("touchstart", (() => {
      let last = 0;
      return (e) => {
        const now = Date.now();
        if (now - last < 400) createMeteor();
        last = now;
      };
    })());

    function createImpactParticles(x, y, count = 20, color = "#FFFFFF") {
        for (let i = 0; i < count; i++) {
            const angle = Math.random() * Math.PI * 2;
            const speed = 2 + Math.random() * 5;
            impactParticles.push({
                x: x,
                y: y,
                vx: Math.cos(angle) * speed,
                vy: Math.sin(angle) * speed,
                alpha: 1,
                size: 1 + Math.random() * 2,
                color: color
            });
        }
        // Vibración al impacto de un meteorito
        if (navigator.vibrate) {
            navigator.vibrate([100, 50, 100]); // Patrón de vibración más pronunciado
        }
    }

    function drawImpactParticles() {
        for (let i = 0; i < impactParticles.length; i++) {
            let p = impactParticles[i];
            ctx.beginPath();
            ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
            ctx.fillStyle = `rgba(, , , )`;
            ctx.fill();

            p.x += p.vx;
            p.y += p.vy;
            p.alpha -= 0.03;
            p.size *= 0.98;
        }
        impactParticles = impactParticles.filter(p => p.alpha > 0 && p.size > 0.1);
    }

    function drawMeteoritos() {
      for (let meteor of meteoritos) {
        meteor.trail.unshift({ x: meteor.x, y: meteor.y });
        if (meteor.trail.length > 30) meteor.trail.pop();

        // Aplica el desplazamiento del sensor a la posición de los meteoritos al dibujarlos
        ctx.save();
        ctx.translate(deviceOrientationX, deviceOrientationY);

        for (let i = 0; i < meteor.trail.length; i++) {
          const t = meteor.trail[i];
          ctx.beginPath();
          ctx.arc(t.x, t.y, 2, 0, Math.PI * 2);
          ctx.fillStyle = `rgba(255,255,255,)`;
          ctx.fill();
        }

        ctx.beginPath();
        ctx.arc(meteor.x, meteor.y, 6, 0, Math.PI * 2);
        ctx.fillStyle = "white";
        ctx.shadowColor = "white";
        ctx.shadowBlur = 15;
        ctx.fill();
        ctx.shadowBlur = 0;
        ctx.restore(); // Restaura la transformación

        meteor.x += meteor.vx;
        meteor.y += meteor.vy;

        const distToHeartCenter = Math.sqrt(Math.pow(meteor.x - w/2, 2) + Math.pow(meteor.y - h/2, 2));
        if (distToHeartCenter < 150 && !meteor.hasImpacted) {
            createImpactParticles(meteor.x, meteor.y, 30, "#FF6699");
            meteor.hasImpacted = true;
        }
      }

      meteoritos = meteoritos.filter(m => m.x > -100 && m.x < w+100 && m.y > -100 && m.y < h+100);
    }


    function animate() {
      ctx.clearRect(0, 0, w, h);
      drawStars();
      drawMeteoritos();
      drawImpactParticles();

      const time = Date.now() * 0.001;

      for (let bot of microbots) {
        let dx = bot.x - mouse.x;
        let dy = bot.y - mouse.y;
        let dist = Math.sqrt(dx*dx + dy*dy);
        if (dist < mouse.radius) {
          let force = (mouse.radius - dist) / mouse.radius;
          let angle = Math.atan2(dy, dx);
          bot.vx += Math.cos(angle) * force * 2.5;
          bot.vy += Math.sin(angle) * force * 2.5;
        }

        for (let meteor of meteoritos) {
          let dx = bot.x - meteor.x;
          let dy = bot.y - meteor.y;
          let d = Math.sqrt(dx*dx + dy*dy);
          if (d < 100) {
            let force = (100 - d) / 100;
            let angle = Math.atan2(dy, dx);
            bot.vx += Math.cos(angle) * force * 5;
            bot.vy += Math.sin(angle) * force * 5;
          }
        }

        const pulseEffect = Math.sin(time * pulseFrequency) * pulseAmplitude;
        
        // --- Aplicar desplazamiento del acelerómetro/giroscopio al punto base del bot ---
        const currentBaseX = bot.initialX + (bot.initialX - w/2) * pulseEffect + deviceOrientationX;
        const currentBaseY = bot.initialY + (bot.initialY - h/2) * pulseEffect + deviceOrientationY;
        // -----------------------------------------------------------------------------

        bot.vx += (currentBaseX - bot.x) * 0.01;
        bot.vy += (currentBaseY - bot.y) * 0.01;
        bot.vx *= 0.88;
        bot.vy *= 0.88;
        bot.x += bot.vx;
        bot.y += bot.vy;

        ctx.beginPath();
        ctx.arc(bot.x, bot.y, 0.8, 0, Math.PI * 2);
        ctx.fillStyle = bot.color;
        ctx.fill();
      }

      requestAnimationFrame(animate);
    }

    animate();
  </script>
</body>
</html>
💗.]()
