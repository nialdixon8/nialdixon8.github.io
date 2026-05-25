---
layout: default
title: Home
---

<!-- About Section -->
<section id="about">
    <h2>About Me</h2>
    <p>Welcome! I'm a software engineer and AI enthusiast with a passion for building reliable and elegant software solutions. With a strong foundation across the stack and a keen interest in artificial intelligence, I'm dedicated to crafting software that solves real-world problems.</p>
    <p>I have professional experience across the full stack, having worked on backend systems built with Java and Spring Boot — with a focus on reliability, performance, and testing — as well as iOS frontend development using Swift and SwiftUI. I enjoy working across different layers of the stack and adapting to new technologies quickly.</p>
</section>

<!-- Game Section -->
<section id="game">
    <h2>Take a Break</h2>
    <p style="color: var(--text-secondary); margin-bottom: 20px;">Arrow keys or WASD to drive — Can you beat your fastest lap.</p>
    <div style="display:flex; flex-direction:column; align-items:center; gap:12px;">
        <div style="display:flex; justify-content:space-between; width:100%; max-width:580px; font-size:13px; color:var(--text-secondary);">
            <span>Lap: <strong id="g-lap">0</strong></span>
            <span>Current: <strong id="g-cur">--:--.--</strong></span>
            <span>Fastest: <strong id="g-best">--:--.--</strong></span>
        </div>
        <canvas id="raceCanvas" width="580" height="400" style="border-radius:12px; border:1px solid var(--border-color); display:block; background:#1a2a1a;"></canvas>
        <p style="font-size:12px; color:var(--text-secondary);">Arrow keys / WASD to steer. Stay on the track!</p>
    </div>
</section>

<script>
(function() {
    const canvas = document.getElementById('raceCanvas');
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    const W = 580, H = 400;

    const trackPts = [
        {x:290, y:340}, {x:100, y:340}, {x:60,  y:280}, {x:60,  y:180},
        {x:110, y:100}, {x:200, y:70},  {x:290, y:80},  {x:370, y:60},
        {x:460, y:80},  {x:510, y:160}, {x:510, y:260}, {x:460, y:320},
        {x:390, y:340}
    ];

    // Zone distances from centreline
    // Road:        -ROAD_W  to  +ROAD_W
    // Inner grass: inner side, ROAD_W out to INNER_WALL (= ROAD_W * 1.75)
    // Outer grass: outer side, ROAD_W out to OUTER_WALL (= ROAD_W * 1.2)
    const ROAD_W      = 24;
    const INNER_GRASS = ROAD_W * 0.75;
    const OUTER_GRASS = ROAD_W * 0.2;
    const INNER_WALL  = ROAD_W + INNER_GRASS;   // 42
    const OUTER_WALL  = ROAD_W + OUTER_GRASS;   // ~29

    function catmullRom(pts, steps) {
        const result = [];
        const n = pts.length;
        for (let i = 0; i < n; i++) {
            const p0=pts[(i-1+n)%n], p1=pts[i], p2=pts[(i+1)%n], p3=pts[(i+2)%n];
            for (let s = 0; s < steps; s++) {
                const t=s/steps, t2=t*t, t3=t2*t;
                const x=0.5*((2*p1.x)+(-p0.x+p2.x)*t+(2*p0.x-5*p1.x+4*p2.x-p3.x)*t2+(-p0.x+3*p1.x-3*p2.x+p3.x)*t3);
                const y=0.5*((2*p1.y)+(-p0.y+p2.y)*t+(2*p0.y-5*p1.y+4*p2.y-p3.y)*t2+(-p0.y+3*p1.y-3*p2.y+p3.y)*t3);
                result.push({x, y});
            }
        }
        return result;
    }

    const smooth = catmullRom(trackPts, 12);
    const N = smooth.length;

    function getNormal(i) {
        const a=smooth[(i-1+N)%N], b=smooth[(i+1)%N];
        const dx=b.x-a.x, dy=b.y-a.y, len=Math.hypot(dx,dy);
        return {x:-dy/len, y:dx/len};
    }
    function getTangent(i) {
        const a=smooth[(i-1+N)%N], b=smooth[(i+1)%N];
        const dx=b.x-a.x, dy=b.y-a.y, len=Math.hypot(dx,dy);
        return {x:dx/len, y:dy/len};
    }

    // Precompute edge curves
    const edgeRoad_O  = smooth.map((p,i)=>{ const n=getNormal(i); return {x:p.x+n.x*ROAD_W,      y:p.y+n.y*ROAD_W};      });
    const edgeRoad_I  = smooth.map((p,i)=>{ const n=getNormal(i); return {x:p.x-n.x*ROAD_W,      y:p.y-n.y*ROAD_W};      });
    const edgeOWall   = smooth.map((p,i)=>{ const n=getNormal(i); return {x:p.x+n.x*OUTER_WALL,  y:p.y+n.y*OUTER_WALL};  });
    const edgeIWall   = smooth.map((p,i)=>{ const n=getNormal(i); return {x:p.x-n.x*INNER_WALL,  y:p.y-n.y*INNER_WALL};  });

    // Find nearest centerline index — search full loop
    function nearestIdx(x, y) {
        let best=0, bd=Infinity;
        for (let i=0; i<N; i++) {
            const d=Math.hypot(smooth[i].x-x, smooth[i].y-y);
            if (d < bd) { bd=d; best=i; }
        }
        return best;
    }

    // Returns signed lateral offset from centreline:
    // positive = outer (right of travel), negative = inner (left of travel / centre island)
    function signedLateral(x, y) {
        const i = nearestIdx(x, y);
        const n = getNormal(i);
        return (x - smooth[i].x)*n.x + (y - smooth[i].y)*n.y;
    }

    function getZone(x, y) {
        const s = signedLateral(x, y);
        if (s >= 0) {
            // Outer side
            if (s <= ROAD_W)     return 'road';
            if (s <= OUTER_WALL) return 'outer-grass';
            return 'outer-wall';
        } else {
            // Inner side
            const d = -s;
            if (d <= ROAD_W)     return 'road';
            if (d <= INNER_WALL) return 'inner-grass';
            return 'inner-wall';
        }
    }

    // ---- Wall collision: use signed lateral to push car correctly ----
    // This avoids the fly-off bug by always pushing along the true normal direction
    function resolveWalls() {
        const i = nearestIdx(car.x, car.y);
        const n = getNormal(i);
        const s = signedLateral(car.x, car.y);

        if (s > OUTER_WALL) {
            // Hit outer wall — push inward (negative normal direction)
            const overlap = s - OUTER_WALL;
            car.x -= n.x * overlap;
            car.y -= n.y * overlap;
            car.speed *= 0.25;
        } else if (s < -INNER_WALL) {
            // Hit inner wall — push outward (positive normal direction)
            const overlap = (-s) - INNER_WALL;
            car.x += n.x * overlap;
            car.y += n.y * overlap;
            car.speed *= 0.25;
        }
    }

    // ---- Drawing ----
    function buildPath(pts) {
        ctx.moveTo(pts[0].x, pts[0].y);
        for (let i=1; i<N; i++) ctx.lineTo(pts[i].x, pts[i].y);
        ctx.closePath();
    }

    function strokeLoop(pts, color, width, dash=[]) {
        ctx.strokeStyle = color; ctx.lineWidth = width; ctx.setLineDash(dash);
        ctx.beginPath(); buildPath(pts); ctx.stroke();
        ctx.setLineDash([]);
    }

    function fillShape(pts, color) {
        ctx.beginPath();
        ctx.moveTo(pts[0].x, pts[0].y);
        for (let i = 1; i < pts.length; i++) ctx.lineTo(pts[i].x, pts[i].y);
        ctx.closePath();
        ctx.fillStyle = color;
        ctx.fill();
    }

    function drawTrack() {
        // Layer 1: green everywhere (outer grass + background)
        ctx.fillStyle = '#4a8a4a';
        ctx.fillRect(0, 0, W, H);

        // Layer 2: paint outer wall as dark band — fill out to edgeOWall, green already showing outside
        // Actually outer wall = hard red line on edgeOWall
        strokeLoop(edgeOWall, '#dd2222', 5);

        // Layer 3: grey road (full blob from edgeRoad_O inward — fills road + inner grass + island)
        fillShape(edgeRoad_O, '#707070');

        // Layer 4: inner grass ring — green band between edgeRoad_I and edgeIWall
        fillShape(edgeRoad_I, '#4a8a4a');

        // Layer 5: inner wall line
        strokeLoop(edgeIWall, '#dd2222', 5);

        // Layer 6: black centre island (inside edgeIWall)
        fillShape(edgeIWall, '#111111');

        // Layer 7: centre dashes on road
        strokeLoop(smooth, 'rgba(255,255,180,0.6)', 1.5, [10, 12]);

        // Layer 8: white kerb lines at road edges
        strokeLoop(edgeRoad_O, 'rgba(255,255,255,0.5)', 1.5, [6, 8]);
        strokeLoop(edgeRoad_I, 'rgba(255,255,255,0.5)', 1.5, [6, 8]);

        // Layer 9: start/finish checkerboard
        const sf = smooth[0], n0 = getNormal(0);
        const sq = 6, steps = Math.floor(ROAD_W * 2 / sq);
        for (let k = 0; k < steps; k++) {
            const t = (k / steps) * 2 * ROAD_W - ROAD_W;
            ctx.fillStyle = k % 2 === 0 ? '#ffffff' : '#111111';
            ctx.fillRect(sf.x + n0.x*t - sq/2, sf.y + n0.y*t - sq/2, sq, sq);
        }
    }

    // ---- Car spawn aligned to track direction ----
    const spawnTangent = getTangent(0);
    const spawnAngle   = Math.atan2(spawnTangent.x, -spawnTangent.y);
    const spawnNormal  = getNormal(0);
    let car = {
        x: smooth[0].x + spawnNormal.x * (ROAD_W * 0.4),
        y: smooth[0].y + spawnNormal.y * (ROAD_W * 0.4),
        angle: spawnAngle,
        speed: 0
    };

    let keys = {};
    let lap=0, lapStart=null, curTime=0, bestLap=null, started=false;
    let prevX=car.x, prevY=car.y, crossCooldown=0;

    // ---- Lap line: segment intersection (only counts forward crossings) ----
    function getSFSeg() {
        const sf=smooth[0], n0=getNormal(0);
        return { x1:sf.x+n0.x*ROAD_W, y1:sf.y+n0.y*ROAD_W,
                 x2:sf.x-n0.x*ROAD_W, y2:sf.y-n0.y*ROAD_W };
    }

    function segsCross(ax,ay,bx,by,cx,cy,dx,dy) {
        const d1x=bx-ax, d1y=by-ay, d2x=dx-cx, d2y=dy-cy;
        const denom = d1x*d2y - d1y*d2x;
        if (Math.abs(denom)<1e-10) return false;
        const t=((cx-ax)*d2y-(cy-ay)*d2x)/denom;
        const u=((cx-ax)*d1y-(cy-ay)*d1x)/denom;
        return t>=0&&t<=1&&u>=0&&u<=1;
    }

    function checkLap() {
        if (crossCooldown > 0) { crossCooldown--; return; }
        const sf = getSFSeg();
        if (car.speed > 0.5 && segsCross(prevX,prevY,car.x,car.y, sf.x1,sf.y1,sf.x2,sf.y2)) {
            if (lapStart !== null) {
                if (bestLap===null || curTime<bestLap) bestLap=curTime;
            }
            lap++;
            lapStart = performance.now();
            curTime = 0;
            crossCooldown = 120;
        }
    }

    function fmtTime(ms) {
        if (ms===null) return '--:--.--';
        const m=Math.floor(ms/60000), s=Math.floor((ms%60000)/1000), h=Math.floor((ms%1000)/10);
        return `${m}:${String(s).padStart(2,'0')}.${String(h).padStart(2,'0')}`;
    }
    function updateHUD() {
        document.getElementById('g-lap').textContent = lap;
        document.getElementById('g-cur').textContent = fmtTime(lapStart!==null ? curTime : null);
        document.getElementById('g-best').textContent = fmtTime(bestLap);
    }

    document.addEventListener('keydown', e => {
        keys[e.key]=true;
        if (['ArrowUp','ArrowDown','ArrowLeft','ArrowRight',' '].includes(e.key)) e.preventDefault();
        if (!started) { started=true; lapStart=performance.now(); }
    });
    document.addEventListener('keyup', e => keys[e.key]=false);

    function drawCar() {
        ctx.save();
        ctx.translate(car.x, car.y);
        ctx.rotate(car.angle);
        ctx.fillStyle = '#00d4ff';
        ctx.beginPath(); ctx.roundRect(-7,-12,14,24,3); ctx.fill();
        ctx.fillStyle = 'rgba(0,0,0,0.5)';
        ctx.fillRect(-4,-9,8,6);
        ctx.fillStyle = '#222';
        [[-9,-8],[5,-8],[-9,5],[5,5]].forEach(([wx,wy])=>ctx.fillRect(wx,wy,4,6));
        ctx.restore();
    }

    let lastTime=null;
    function loop(ts) {
        if (!lastTime) lastTime=ts;
        const dt = Math.min((ts-lastTime)/16.67, 3);
        lastTime=ts;
        if (lapStart!==null) curTime = ts-lapStart;

        const accel = keys['ArrowUp']    || keys['w'] || keys['W'];
        const brake = keys['ArrowDown']  || keys['s'] || keys['S'];
        const left  = keys['ArrowLeft']  || keys['a'] || keys['A'];
        const right = keys['ArrowRight'] || keys['d'] || keys['D'];

        const zone   = getZone(car.x, car.y);
        const onRoad = zone === 'road';
        const onGrass= zone === 'inner-grass' || zone === 'outer-grass';
        const grip   = onRoad ? 1 : 0.45;
        const maxSpd = onRoad ? 3.5 : 1.6;

        if (accel) car.speed = Math.min(car.speed + 0.06*grip*dt, maxSpd);
        else if (brake) car.speed = Math.max(car.speed - 0.1*dt, -1.2);
        else car.speed *= Math.pow(0.97, dt);

        if (Math.abs(car.speed) > 0.1) {
            const steerAmt = 0.045*(Math.abs(car.speed)/3.5)*grip*dt;
            if (left)  car.angle -= steerAmt;
            if (right) car.angle += steerAmt;
        }

        prevX=car.x; prevY=car.y;
        car.x += Math.sin(car.angle)*car.speed*dt;
        car.y -= Math.cos(car.angle)*car.speed*dt;

        resolveWalls();
        checkLap();

        drawTrack();
        drawCar();

        if (!started) {
            ctx.fillStyle='rgba(0,0,0,0.6)'; ctx.fillRect(0,0,W,H);
            ctx.fillStyle='#fff'; ctx.font='500 20px sans-serif'; ctx.textAlign='center';
            ctx.fillText('Top Down Racer', W/2, H/2-20);
            ctx.font='400 13px sans-serif'; ctx.fillStyle='rgba(255,255,255,0.6)';
            ctx.fillText('Press any key to start — Arrow keys / WASD', W/2, H/2+10);
        }

        if (started && onGrass) {
            ctx.fillStyle='rgba(255,80,30,0.12)'; ctx.fillRect(0,0,W,H);
            ctx.fillStyle='#ffaa44'; ctx.font='500 12px sans-serif'; ctx.textAlign='center';
            ctx.fillText('Grass — slow!', W/2, 20);
        }

        updateHUD();
        requestAnimationFrame(loop);
    }

    requestAnimationFrame(loop);
})();
</script>

<!-- Education Section -->
<section id="education">
    <h2>Education</h2>
    <div class="timeline">
        <div class="timeline-item">
            <div class="timeline-title">MSc Advanced Computer Science</div>
            <div class="timeline-date">University of Birmingham • Sep 2024 – Sep 2025</div>
            <div class="timeline-description">Advancing expertise in computer science with focus on Distributed Systems, High Performance Computing and Advanced Algorithms.</div>
        </div>
        <div class="timeline-item">
            <div class="timeline-title">BSc Software Engineering</div>
            <div class="timeline-date">University of Leicester • Sep 2020 – May 2024</div>
            <div class="timeline-description">First Class Honours. Strong foundation in software development, systems design, and computer science fundamentals.</div>
        </div>
    </div>
</section>

<!-- Skills Section -->
<section id="skills">
    <h2>Skills & Technologies</h2>
    <p>I work across the full technology stack with expertise in:</p>
    <div class="skills-container">
        <span class="skill-badge">Python</span>
        <span class="skill-badge">JavaScript/TypeScript</span>
        <span class="skill-badge">React</span>
        <span class="skill-badge">Node.js</span>
        <span class="skill-badge">Machine Learning</span>
        <span class="skill-badge">PyTorch</span>
        <span class="skill-badge">TensorFlow</span>
        <span class="skill-badge">SQL</span>
        <span class="skill-badge">MongoDB</span>
        <span class="skill-badge">AWS</span>
        <span class="skill-badge">Docker</span>
        <span class="skill-badge">Git</span>
        <span class="skill-badge">REST APIs</span>
        <span class="skill-badge">Data Analysis</span>
        <span class="skill-badge">NLP</span>
        <span class="skill-badge">Computer Vision</span>
    </div>
</section>

<!-- Experience Section -->
<section id="experience">
    <h2>Professional Experience</h2>
    <div class="timeline">
        <div class="timeline-item">
            <div class="timeline-title">Software Engineer – iOS Frontend</div>
            <div class="timeline-date">Lloyds Banking Group • May 2026 – Present</div>
            <div class="timeline-description">Recently Started</div>
        </div>
        <div class="timeline-item">
            <div class="timeline-title">Software Engineer – Backend</div>
            <div class="timeline-date">Lloyds Banking Group • Sep 2025 – May 2026</div>
            <div class="timeline-description">Built and maintained backend services using Java and Spring Boot, with a strong focus on testing, reliability, and latency optimisation to ensure high-performance and resilient banking systems.</div>
        </div>
        <div class="timeline-item">
            <div class="timeline-title">Mathematics & Computing Tutor</div>
            <div class="timeline-date">Oct 2020 – Aug 2025</div>
            <div class="timeline-description">Delivering tailored tutoring in mathematics and computer science, enhancing academic performance and problem-solving skills for students of various levels.</div>
        </div>
        <div class="timeline-item">
            <div class="timeline-title">Developer Analyst</div>
            <div class="timeline-date">Barclays • Jun 2022 – Aug 2022</div>
            <div class="timeline-description">Led infrastructure transition between technologies, reducing annual licensing costs. Demonstrated and evaluated emerging AI technologies to business units, translating technical capabilities into business value.</div>
        </div>
    </div>
</section>

<!-- Projects Section -->
<section id="projects">
    <h2>Featured Projects</h2>
    <div class="card-grid">
        <div class="card">
            <h3>AI Stock Prediction Application</h3>
            <p>A comprehensive full-stack AI application that enables users to build and maintain advanced stock prediction algorithms without coding.</p>
            <div class="card-meta">
                <strong>Tech Stack:</strong> React, Redux, Node.js, MongoDB, PyTorch
            </div>
            <p style="margin-top: 15px; font-size: 0.9rem;">Built a sophisticated machine learning pipeline that processes historical data and market indicators to generate predictions for personal investment strategies.</p>
        </div>
        <div class="card">
            <h3>Crypto Sentiment Analytics</h3>
            <p>A startup project analyzing cryptocurrency market sentiment and correlating it with price movements using advanced NLP and statistical analysis.</p>
            <div class="card-meta">
                <strong>Tech Stack:</strong> Python, LLMs, Statistical Analysis, Web Scraping
            </div>
            <p style="margin-top: 15px; font-size: 0.9rem;">Engineered sophisticated data collection tools to gather articles and opinions from multiple media sources, then applied LLM-based sentiment analysis to identify market correlations.</p>
        </div>
        <div class="card">
            <h3>Raft in Rust</h3>
            <p>A complete Raft consensus algorithm implementation written in Rust.</p>
            <div class="card-meta">
                <strong>Tech Stack:</strong> Rust, Tokio
            </div>
            <p style="margin-top: 15px; font-size: 0.9rem;">
                A fault-tolerant distributed consensus system implementing the Raft protocol, 
                featuring leader election and replicated log management across a cluster of nodes. 
                Includes a CLI for interacting with the cluster directly and an API for programmatic 
                integration into distributed applications.
            </p>
        </div>
    </div>
</section>

<!-- Contact Section -->
<section id="contact">
    <h2>Let's Connect</h2>
    <p>I'm always interested in discussing technology, feel free to reach out!</p>
    <div class="hero-cta" style="margin-top: 30px;">
        <a href="https://github.com/nialdixon8" target="_blank" class="btn btn-primary">GitHub</a>
        <a href="https://linkedin.com/in/nial-dixon" target="_blank" class="btn btn-primary">LinkedIn</a>
        <a href="mailto:nialdixon8@gmail.com" class="btn btn-secondary">Email Me</a>
    </div>
</section>