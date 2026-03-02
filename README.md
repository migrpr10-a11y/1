// --- PROJECT: X_MIXER | CORE LOGIC ---

// 1. Configuração do Canvas para o Gráfico
const canvas = document.getElementById('audio-graph');
const canvasCtx = canvas.getContext('2d');

// Ajusta o tamanho do canvas para o tamanho real do elemento
function resizeCanvas() {
    canvas.width = canvas.parentElement.clientWidth;
    canvas.height = canvas.parentElement.clientHeight;
}
window.addEventListener('resize', resizeCanvas);
resizeCanvas(); // Chama no início

// 2. Configuração de Áudio (Web Audio API)
let audioCtx;
let source;
let analyser;
let bufferLength;
let dataArray;
let gainNode; // Para controlar o volume (808 Dist)

// Mapeamento de Gêneros para arquivos de som
// !!! IMPORTANTE: Você precisa criar ou baixar esses arquivos .mp3 !!!
const genreTracks = {
    rage: 'rage_beat.mp3',
    darktrap: 'darktrap_beat.mp3',
    plugg: 'plugg_beat.mp3',
    ambient: 'ambient_beat.mp3'
};

let currentGenre = null;

// 3. Função para Iniciar/Trocar o Áudio
async function playGenre(genre) {
    // Inicia o contexto de áudio se não existir (precisa de um clique do usuário)
    if (!audioCtx) {
        audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    }

    // Para o som atual se houver
    if (source) {
        source.stop();
    }

    currentGenre = genre;
    const filePath = genreTracks[genre];

    try {
        const response = await fetch(filePath);
        if (!response.ok) throw new Error(`Não foi possível carregar: ${filePath}`);
        const arrayBuffer = await response.arrayBuffer();
        const audioBuffer = await audioCtx.decodeAudioData(arrayBuffer);

        source = audioCtx.createBufferSource();
        source.buffer = audioBuffer;
        source.loop = true; // Faz o beat tocar em loop

        // Cria o Analisador para o gráfico
        analyser = audioCtx.createAnalyser();
        analyser.fftSize = 2048; // Tamanho do processamento de áudio
        bufferLength = analyser.frequencyBinCount;
        dataArray = new Uint8Array(bufferLength);

        // Cria o GainNode para o slider "808 DIST" (simulado como volume aqui)
        gainNode = audioCtx.createGain();
        const distSlider = document.querySelector('.slider-neon');
        gainNode.gain.value = distSlider.value / 100; // Define o volume inicial do slider

        // Conecta tudo: Som -> Dist (Volume) -> Analisador -> Saída Final
        source.connect(gainNode);
        gainNode.connect(analyser);
        analyser.connect(audioCtx.destination);

        source.start(0);
        console.log(`Tocando: ${genre}`);

        // Inicia o desenho do gráfico
        drawGraph();

    } catch (error) {
        console.error("Erro de áudio:", error);
        alert(`Erro: Você precisa adicionar o arquivo '${filePath}' na mesma pasta para ouvir o som.`);
    }
}

// 4. Função para Desenhar o Gráfico (Onda Verde Neon)
function drawGraph() {
    // Para a animação se não houver áudio
    if (!analyser || !currentGenre) return;

    requestAnimationFrame(drawGraph); // Cria o loop de animação

    analyser.getByteTimeDomainData(dataArray); // Pega os dados da onda sonora

    // Limpa o canvas com o fundo dark
    canvasCtx.fillStyle = '#080808';
    canvasCtx.fillRect(0, 0, canvas.width, canvas.height);

    // Configura o estilo da linha da onda (Neon Green)
    canvasCtx.lineWidth = 2;
    canvasCtx.strokeStyle = '#00ff41'; // --neon-green
    canvasCtx.shadowBlur = 10;
    canvasCtx.shadowColor = '#00ff41'; // Efeito de brilho neon

    canvasCtx.beginPath();

    const sliceWidth = canvas.width * 1.0 / bufferLength;
    let x = 0;

    for (let i = 0; i < bufferLength; i++) {
        const v = dataArray[i] / 128.0; // Normaliza os dados
        const y = v * canvas.height / 2; // Centraliza a onda verticalmente

        if (i === 0) {
            canvasCtx.moveTo(x, y);
        } else {
            canvasCtx.lineTo(x, y);
        }

        x += sliceWidth;
    }

    canvasCtx.lineTo(canvas.width, canvas.height / 2);
    canvasCtx.stroke();
    // Reseta o blur para não afetar outros desenhos (boa prática)
    canvasCtx.shadowBlur = 0;
}

// 5. Event Listeners (Controles da Interface)

// Botões de Gênero
const genreButtons = document.querySelectorAll('.genre-btn');
genreButtons.forEach(button => {
    button.addEventListener('click', () => {
        // Remove a classe 'active' de todos
        genreButtons.forEach(btn => btn.classList.remove('active'));
        // Adiciona ao botão clicado
        button.classList.add('active');

        const genre = button.getAttribute('data-genre');
        playGenre(genre);
    });
});

// Slider "808 DIST" (Volume)
const distSlider = document.querySelector('.slider-neon');
distSlider.addEventListener('input', () => {
    if (gainNode) {
        // Mapeia o valor do slider (0-100) para o volume do GainNode (0.0-1.0)
        gainNode.gain.value = distSlider.value / 100;
    }
});

// Botão Master "APPLY_MIX_CHAOS" (simula uma ação)
const masterBtn = document.getElementById('master-btn');
masterBtn.addEventListener('click', () => {
    if (currentGenre) {
        masterBtn.innerText = "MIX_APPLIED::SUCCESS";
        masterBtn.style.borderColor = "#fff";
        masterBtn.style.color = "#fff";
        setTimeout(() => {
            masterBtn.innerText = "APPLY_MIX_CHAOS";
            masterBtn.style.borderColor = "#00ff41";
            masterBtn.style.color = "#00ff41";
        }, 2000);
    } else {
        alert("Escolha um gênero primeiro.");
    }
});
