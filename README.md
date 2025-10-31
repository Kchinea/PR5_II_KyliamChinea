# PR5_II_KyliamChinea

## Índice

- [Introducción](#Introducción)
- [Códigos](#Código)
- [GamePlay](#GamePlay)

---
## Introducción

A la hora de realizar esta practica comence desde el tutorial que se realizó en clase, a partir del script de `Treasure` y con el patrón observador realice el script `ObjectController`, `Reaction` y `Notifier`,  utilice assets de navidad y cree un entorno en el que al ejecutarse el metodo `OnPointerEnter` se recolecta el objeto (pingüino) y al mirar al árbol de navidad gigante aparecen los pingüinos recolectados y van hacia el jugador, al llegar se ponen a saltar, una de las principales dificultades a la hora de realizar la practica, fue que al aparecer los pingüinos chocaban entre si al aparecer juntos y salian volando, or ello decidí desactivarle las fisícas a la hora de aparecer y volverlas a activar pasado un tiempo corto.

## Código

### ObjectController
```csharp
using UnityEngine;
using System.Collections;

[RequireComponent(typeof(Collider))]
public class CollectibleController : MonoBehaviour
{
    [Tooltip("Distancia máxima para poder recoger con la mirada")]
    public float collectDistance = 40f;

    [Tooltip("Tag del objeto notificador de recuperación")]
    public string notifierTag = "Recovery Object";

    [HideInInspector] public Vector3 originalPosition;
    [HideInInspector] public Quaternion originalRotation;

    private GameObject notifierObject;
    private RecoveryNotifier notifier;
    private bool isCollected = false;
    private bool isRecovering = false;

    void Start()
    {
        originalPosition = transform.position;
        originalRotation = transform.rotation;

        // Buscar y suscribirse al notificador
        notifierObject = GameObject.FindWithTag(notifierTag);
        if (notifierObject != null)
        {
            notifier = notifierObject.GetComponent<RecoveryNotifier>();
            if (notifier != null)
            {
                notifier.OnPlayerGaze += OnRecoverTriggered;
            }
        }
    }

    void OnDestroy()
    {
        if (notifier != null)
        {
            notifier.OnPlayerGaze -= OnRecoverTriggered;
        }
    }

    public void OnPointerEnter()
    {
        if (isCollected || isRecovering) return;

        GameObject player = GameObject.FindWithTag("Player");
        if (player == null) return;
        
        float distance = Vector3.Distance(player.transform.position, transform.position);
        if (distance <= collectDistance && !isCollected)
        {
            isCollected = true;
            gameObject.SetActive(false);
        }
    }

    void OnRecoverTriggered()
    {
        if (isCollected && !isRecovering)
        {
            StartRecovery();
        }
    }

    void StartRecovery()
    {
        isRecovering = true;

        // Posición aleatoria alrededor del notifier
        if (notifierObject != null)
        {
            float angle = Random.Range(0f, 360f) * Mathf.Deg2Rad;
            float radius = Random.Range(0.5f, 1.5f);
            Vector3 randomOffset = new Vector3(Mathf.Cos(angle) * radius, 0f, Mathf.Sin(angle) * radius);
            
            Vector3 spawnPosition = notifierObject.transform.position + randomOffset;
            spawnPosition.y = 1f; // Y obligatoria en 1 para evitar atravesar el suelo
            transform.position = spawnPosition;
            transform.rotation = Quaternion.identity;
        }

        gameObject.SetActive(true);

        // Desactivar física temporalmente
        Rigidbody rb = GetComponent<Rigidbody>();
        if (rb != null)
        {
            rb.isKinematic = true;
            rb.linearVelocity = Vector3.zero;
            rb.angularVelocity = Vector3.zero;
        }

        // Activar RecoveryMover
        RecoveryMover mover = GetComponent<RecoveryMover>();
        if (mover == null)
        {
            mover = gameObject.AddComponent<RecoveryMover>();
        }
        mover.notifierTransform = notifierObject.transform;
        mover.enabled = true;
        
        StartCoroutine(EnablePhysicsAfterDelay(0.5f));
    }
    
    IEnumerator EnablePhysicsAfterDelay(float delay)
    {
        yield return new WaitForSeconds(delay);
        
        Rigidbody rb = GetComponent<Rigidbody>();
        if (rb != null)
        {
            rb.isKinematic = false;
        }
    }
}


```

### Reaction

```csharp
using UnityEngine;

public class RecoveryMover : MonoBehaviour
{
    [Header("Referencias")]
    public Transform notifierTransform;
    public string playerTag = "Player";

    [Header("Movimiento")]
    public float speed = 8f;
    public float stopDistance = 2f;
    public float arriveDistance = 0.5f;

    [Header("Saltos")]
    public float jumpHeight = 0.8f;
    public float jumpSpeed = 3f;

    private Transform player;
    private Rigidbody rb;
    private bool arrived = false;
    private Vector3 jumpBasePosition;
    private float jumpOffset;

    void Start()
    {
        player = GameObject.FindWithTag(playerTag)?.transform;
        rb = GetComponent<Rigidbody>();
        jumpOffset = Random.Range(0f, Mathf.PI * 2f);
    }

    void OnEnable()
    {
        arrived = false;
        if (player == null)
            player = GameObject.FindWithTag(playerTag)?.transform;
        jumpOffset = Random.Range(0f, Mathf.PI * 2f);
    }

    void Update()
    {
        if (player == null)
        {
            player = GameObject.FindWithTag(playerTag)?.transform;
            if (player == null) return;
        }

        // Solo mover si el Rigidbody está en modo kinematic (física desactivada)
        if (rb != null && !rb.isKinematic) return;

        if (!arrived)
        {
            Vector3 playerPos = player.position;
            Vector3 direction = (playerPos - transform.position).normalized;
            Vector3 targetPos = playerPos - direction * stopDistance;
            
            float dist = Vector3.Distance(transform.position, targetPos);

            if (dist <= arriveDistance)
            {
                arrived = true;
                jumpBasePosition = transform.position;
                return;
            }

            transform.position = Vector3.MoveTowards(transform.position, targetPos, speed * Time.deltaTime);

            Vector3 lookDir = (player.position - transform.position);
            if (lookDir.sqrMagnitude > 0.001f)
                transform.rotation = Quaternion.Slerp(transform.rotation, Quaternion.LookRotation(lookDir.normalized), 10f * Time.deltaTime);
        }
        else
        {
            float jumpY = Mathf.Abs(Mathf.Sin(Time.time * jumpSpeed + jumpOffset)) * jumpHeight;
            transform.position = jumpBasePosition + Vector3.up * jumpY;
        }
    }
}

```

### Notifier

```csharp
using UnityEngine;

public class RecoveryNotifier : MonoBehaviour
{
    // Delegate y evento para notificar a los collectibles
    public delegate void RecoverDelegate();
    public event RecoverDelegate OnPlayerGaze;

    [Header("Fallback gaze detection (raycast to collider)")]
    [Tooltip("Enable fallback gaze detection using a Physics.Raycast from the camera center to hit this object's collider.")]
    public bool useRaycastFallback = true;

    [Tooltip("Maximum distance (meters) for the fallback gaze raycast")]
    public float gazeMaxDistance = 100f;

    [Tooltip("Time (seconds) the player must hold gaze to trigger recovery when using fallback")]
    public float dwellTime = 0.5f;

    [Tooltip("Layer mask used by the fallback raycast. Use this to limit which colliders are considered.")]
    public LayerMask gazeLayerMask = ~0;

    [Tooltip("If true, draw the debug ray in the Scene view/Game view (Gizmos) while checking gaze")]
    public bool debugDrawRay = false;

    [Tooltip("Optional camera to use for gaze checks. If null, Camera.main will be used.")]
    public Camera gazeCamera;

    // estado interno del temporizador de permanencia
    float _gazeTimer = 0f;
    float _lastNotifyTime = -10f;
    [Tooltip("Cooldown after a notification (seconds) to avoid double triggers")]
    public float notifyCooldown = 0.5f;

    void Reset()
    {
        // valores predeterminados
        useRaycastFallback = true;
        gazeMaxDistance = 100f;
        dwellTime = 0.5f;
        notifyCooldown = 0.5f;
    }

    void Start()
    {
        if (gazeCamera == null) gazeCamera = Camera.main;
    }

    void Update()
    {
        // Si se le ha notificado recientemente, se añade un período de espera.
        if (Time.time - _lastNotifyTime < notifyCooldown) return;

        if (!useRaycastFallback) return;

        if (gazeCamera == null)
        {
            gazeCamera = Camera.main;
            if (gazeCamera == null) return;
        }
        // Lanzamiento de rayo desde la cámara hacia adelante (punto del centro de la pantalla). Si impacta el colisionador de este objeto (o un hijo), inicia el tiempo de permanencia.
        Vector3 camPos = gazeCamera.transform.position;
        Vector3 camForward = gazeCamera.transform.forward;

        if (debugDrawRay)
        {
            Debug.DrawRay(camPos, camForward * gazeMaxDistance, Color.cyan);
        }

        RaycastHit hit;
        bool hitThis = false;
        if (Physics.Raycast(camPos, camForward, out hit, gazeMaxDistance, gazeLayerMask))
        {
            if (hit.collider != null)
            {
                // Considérese un acierto si el transform impactado es este transform o un derivado de el.
                if (hit.collider.transform == transform || hit.collider.transform.IsChildOf(transform))
                {
                    hitThis = true;
                }
            }
        }

        if (hitThis)
        {
            _gazeTimer += Time.deltaTime;
            if (_gazeTimer >= dwellTime)
            {
                NotifyRecover();
                _gazeTimer = 0f;
            }
        }
        else
        {
            _gazeTimer = 0f;
        }
    }

    /// <summary>
    /// Activado por el sistema de mirada (retícula) cuando se observa el objeto.
    /// </summary>
    public void OnPointerEnter()
    {
        NotifyRecover();
    }

    void NotifyRecover()
    {
        Debug.Log("RecoveryNotifier: Notificando recuperación a todos los suscriptores");
        _lastNotifyTime = Time.time;
        
        // Invocar el evento para notificar a todos los collectibles suscritos
        OnPlayerGaze?.Invoke();
    }
}


```

---

## GamePlay

![GamePlay](./GamePlay.gif)
