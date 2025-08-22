<!DOCTYPE html>
<html>
<head>
  <title>Enviar Vídeo</title>
</head>
<body>
  <h2>Compartilhando câmera...</h2>
  <video id="webcamVideo" autoplay playsinline muted></video>

  <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-firestore-compat.js"></script>
  <script>
    // 🔁 Cole aqui sua configuração do Firebase
    const firebaseConfig = {
      apiKey: "SUA_API_KEY",
      authDomain: "SEU_PROJETO.firebaseapp.com",
      projectId: "SEU_PROJETO",
      storageBucket: "SEU_PROJETO.appspot.com",
      messagingSenderId: "XXXXXXXXXXXX",
      appId: "1:XXXXXXXXXXXX:web:XXXXXXXXXXXX"
    };

    firebase.initializeApp(firebaseConfig);
    const firestore = firebase.firestore();

    const webcamVideo = document.getElementById('webcamVideo');
    const servers = { iceServers: [{ urls: 'stun:stun.l.google.com:19302' }] };
    const pc = new RTCPeerConnection(servers);

    // Obter câmera
    navigator.mediaDevices.getUserMedia({ video: true, audio: false }).then(stream => {
      webcamVideo.srcObject = stream;
      stream.getTracks().forEach(track => pc.addTrack(track, stream));
    });

    // Criar oferta
    const callDoc = firestore.collection('calls').doc('stream');
    const offerCandidates = callDoc.collection('offerCandidates');
    const answerCandidates = callDoc.collection('answerCandidates');

    pc.onicecandidate = event => {
      if (event.candidate) {
        offerCandidates.add(event.candidate.toJSON());
      }
    };

    pc.onconnectionstatechange = () => {
      console.log("Conexão:", pc.connectionState);
    };

    // Criar oferta
    pc.createOffer().then(offer => {
      pc.setLocalDescription(offer);
      callDoc.set({ offer });
    });

    // Escutar resposta
    callDoc.onSnapshot(snapshot => {
      const data = snapshot.data();
      if (!pc.currentRemoteDescription && data?.answer) {
        pc.setRemoteDescription(new RTCSessionDescription(data.answer));
      }
    });

    // Receber candidatos do receptor
    answerCandidates.onSnapshot(snapshot => {
      snapshot.docChanges().forEach(change => {
        if (change.type === 'added') {
          const candidate = new RTCIceCandidate(change.doc.data());
          pc.addIceCandidate(candidate);
        }
      });
    });
  </script>
</body>
</html>
