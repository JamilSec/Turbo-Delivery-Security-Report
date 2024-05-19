# Informe de Vulnerabilidad y Reversión del Código en el Juego Turbo Delivery

**Autor**: Jamil Rodriguez  
**Fecha**: 19 de mayo de 2024  
**Ciudad**: Trujillo - Perú

## Tabla de Contenidos
1. [Introducción](#introducción)
2. [Código Original](#código-original)
3. [Descripción de la Vulnerabilidad](#descripción-de-la-vulnerabilidad)
4. [Proceso de Reversión](#proceso-de-reversión)
5. [Implicaciones de Seguridad](#implicaciones-de-seguridad)
6. [Recomendaciones](#recomendaciones)
7. [Conclusión](#conclusión)

## Introducción

La seguridad en aplicaciones web y juegos en línea es crucial. Este informe analiza una vulnerabilidad en el juego "Turbo Delivery", donde la encriptación del lado del cliente permite manipular puntajes. Se detalla el proceso de reversión del código y se ofrecen recomendaciones para mitigar este riesgo, mejorando la integridad y equidad del juego.

## Código Original

El siguiente fragmento de código corresponde al juego "Turbo Delivery", el cual se ejecuta cuando el documento se ha cargado completamente:

```javascript
document.addEventListener('DOMContentLoaded', () => {
    const modalEndgame = new bootstrap.Modal(document.querySelector('#base-modal'));
    const continueButton = document.querySelector('#btn-continue');
    const closeModal = document.querySelector('#cross-modal-close');
    const btnSubscribe = document.querySelector('#btn-subscribe');
    const btnLogin = document.querySelector('#btn-login');

    const GAME_PUBLIC_KEY = document.querySelector('#public-key').value;

    closeModal.addEventListener('click', onModalClose);
    continueButton.addEventListener('click', onModalClose);

    btnSubscribe.style.display = 'none';
    btnLogin.style.display = 'none';

    continueButton.classList.remove('modal-button--outlined');
    continueButton.classList.add('modal-button--fancy');

    if (window.TurboDelivery) {
        window.TurboDelivery.run({
            onGameEnd,
            onGameStart,
            sponsor: true
        });
    }

    async function onGameEnd(evt) {
        const { score } = evt;
        const isLoggedIn = !!localStorage.getItem('userid');

        const payload = {
            score,
            game_id: document.querySelector('#phaser-div').getAttribute('data-game-id'),
            season_id: document.querySelector('#phaser-div').getAttribute('data-season-id')
        };

        const payloadStr = JSON.stringify(payload);
        const payloadEncrypted = encrypt(payloadStr);

        const threshold = document.querySelector('.main-game-container').getAttribute('data-game-threshold');
        const thresholdAbove = score >= threshold;

        try {
            const { data } = await axios.post('/saveScore', { payload: payloadEncrypted });

            if (!data.leaderboard) {
                return;
            }

            document.querySelector('#endgame-score').textContent = `${isNaN(Number(score)) ? '0' : Number(score).toLocaleString('en-US')} pts`;
            document.querySelector('#endgame-message').innerHTML = 'Próximamente anunciaremos novedades con nuestros juegos.';

            if (thresholdAbove) {
                document.querySelector('#endgame-title').innerHTML = '¡Excelente puntaje!';
            }

            document.querySelector('.leaderboard-mobile').innerHTML = data.leaderboard;
            document.querySelector('.leaderboard-desktop').innerHTML = data.leaderboard;

            modalEndgame.show();
        } catch (error) {
            const { response } = error;

            if (error instanceof Error) {
                console.log(`[Error on onGameEnd function ${error.name}]: ${error.message}`);
            }

            if (response.status === 401) {
                btnSubscribe.style.display = 'block';
                btnLogin.style.display = 'block';
                document.querySelector('#endgame-score').textContent = `${isNaN(Number(score)) ? '0' : Number(score).toLocaleString('en-US')} pts`;

                continueButton.classList.remove('modal-button--fancy');
                continueButton.classList.add('modal-button--outlined');

                document.querySelector('#endgame-title').innerHTML = 'Loguéate para <br /> guardar tu puntaje';
                document.querySelector('#endgame-message').innerHTML = '¡Recuerda!, los suscriptores participan por increíbles premios.';
                modalEndgame.show();
            }
        }

        gtag("event", "game_finished", {
            status: "success",
            gameName: "turbo delivery",
            isLoggedIn,
            thresholdAbove
        });
    }

    function onGameStart() {
        const isLoggedIn = !!localStorage.getItem('userid');

        gtag("event", "game_started", {
            status: "success",
            gameName: "turbo delivery",
            isLoggedIn,
        });
    }

    function encrypt(data) {
        const encrypt = new JSEncrypt();
        encrypt.setPublicKey(GAME_PUBLIC_KEY);
        return encrypt.encrypt(data);
    }

    function onModalClose() {
        modalEndgame.hide();
    }
});
```

## Descripción de la Vulnerabilidad

El juego "Turbo Delivery" utiliza una clave pública para encriptar los datos del puntaje (score) antes de enviarlos al servidor. Sin embargo, debido a que la clave pública y el proceso de encriptación se realizan en el lado del cliente, un atacante con conocimientos básicos de programación y acceso al código puede interceptar y modificar el puntaje antes de que sea encriptado y enviado al servidor.

## Proceso de Reversión

El siguiente código ilustra cómo un atacante podría interceptar y modificar el puntaje:

```javascript
// Constantes y configuración
const GAME_PUBLIC_KEY = document.querySelector('#public-key').value;
const USER_ID = localStorage.getItem('userid');

// Función para encriptar datos
function encrypt(data) {
    const encrypt = new JSEncrypt();
    encrypt.setPublicKey(GAME_PUBLIC_KEY);
    return encrypt.encrypt(data);
}

// Crear el payload con el puntaje modificado
const payloadGame = {
    score: 1000, // Puntaje modificado
    game_id: "1",
    season_id: "2"
};

// Convertir el payload a JSON y encriptarlo
const payloadStr = JSON.stringify(payloadGame);
const encryptedPayload = encrypt(payloadStr);

// Imprimir el resultado en la consola
console.log({ payload: encryptedPayload });

```

En este código, el puntaje (score) se ha modificado a un valor arbitrario antes de ser encriptado. Al imprimir el payload encriptado en la consola, el atacante puede entonces utilizar este valor para enviarlo manualmente al servidor y alterar su posición en el ranking del juego.

## Implicaciones de Seguridad

La vulnerabilidad identificada permite a los jugadores malintencionados:

- Manipular el puntaje para ganar el sorteo de manera fraudulenta.
- Afectar la integridad del juego y la equidad para todos los participantes.
- Poner en riesgo la reputación del juego y la confianza de los usuarios.

## Recomendaciones

Para mitigar esta vulnerabilidad, se sugieren las siguientes medidas:

### Encriptación del lado del servidor

Realizar la encriptación y validación de los datos del puntaje en el servidor, en lugar de hacerlo en el cliente. Esto asegura que los datos enviados no pueden ser manipulados por el usuario.

**Proceso:**
1. El cliente envía los datos sin encriptar al servidor.
2. El servidor valida la autenticidad y exactitud de los datos.
3. El servidor encripta los datos antes de almacenarlos o enviarlos para su procesamiento.

### Validaciones adicionales en el servidor

Implementar validaciones adicionales en el servidor para detectar comportamientos anómalos, como puntajes inusualmente altos o fuera de los límites esperados.

**Proceso:**
1. El servidor mantiene un registro de los puntajes posibles basados en las reglas del juego.
2. Al recibir un nuevo puntaje, el servidor lo compara contra este registro para asegurarse de que es válido.
3. Si el puntaje no es válido, se rechaza y se notifica al usuario.

### Mecanismos de integridad

Utilizar mecanismos de integridad como firmas digitales para asegurar que los datos no hayan sido alterados.

**Proceso:**
1. Al finalizar el juego, el cliente genera un hash de los datos del juego y lo envía junto con los datos al servidor.
2. El servidor recalcula el hash a partir de los datos recibidos y lo compara con el hash enviado.
3. Si los hashes coinciden, se confirma que los datos no han sido alterados; de lo contrario, se rechaza la solicitud.

## Conclusión
La identificación de esta vulnerabilidad demuestra la importancia de manejar de manera segura los datos sensibles en las aplicaciones web. La encriptación del lado del cliente debe ser utilizada con cautela y siempre complementada con medidas de seguridad adicionales en el servidor.

Implementando las recomendaciones mencionadas, es posible mejorar la seguridad y robustez del juego "Turbo Delivery", garantizando una experiencia justa y segura para todos los jugadores. La pasión por la seguridad informática nos motiva a identificar y resolver estas vulnerabilidades para crear un entorno digital más seguro para todos.
