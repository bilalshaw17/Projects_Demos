### Unity 3D Mobile Endless Runner – Complete C# Script Collection

Below are all scripts you need to assemble a basic, performant, lane-based 3D endless runner in Unity (mobile-ready). Create one `.cs` file per section in your Unity `Assets/Scripts` folder with the exact class names and attach them to the appropriate GameObjects as noted in comments.

---

## RunnerConfig.cs

```csharp
using UnityEngine;

namespace EndlessRunner
{
    public static class RunnerConfig
    {
        // Track/Lane
        public const int LaneCount = 3;              // Left, Center, Right
        public const float LaneSpacing = 2.5f;       // Horizontal distance between lanes

        // Motion
        public const float Gravity = -30f;
        public const float InitialForwardSpeed = 8.0f;
        public const float MaxForwardSpeed = 22.0f;
        public const float ForwardAccelerationPerSecond = 0.25f;

        // Spawning
        public const float VisibleDistanceAhead = 120f;
        public const float RecycleDistanceBehind = 40f;

        // Input
        public const float SwipeMinDistancePixels = 75f;
        public const float SwipeMaxTimeSeconds = 0.6f;

        // Coins & Powerups
        public const float MagnetRadius = 6.0f;
        public const float MagnetAttractSpeed = 20.0f;
    }
}
```

---

## GameManager.cs

```csharp
using System;
using UnityEngine;

namespace EndlessRunner
{
    public enum GameState
    {
        Menu,
        Playing,
        Paused,
        GameOver
    }

    public class GameManager : MonoBehaviour
    {
        public static GameManager Instance { get; private set; }

        public event Action GameStarted;
        public event Action GamePaused;
        public event Action GameResumed;
        public event Action GameOverEvent;

        [Header("References")]
        [SerializeField] private Transform playerTransform;
        [SerializeField] private LevelGenerator levelGenerator;

        [Header("Runtime")] 
        [SerializeField] private GameState state = GameState.Menu;

        public GameState State => state;
        public bool IsPlaying => state == GameState.Playing;

        public float RunSpeed { get; private set; }
        public float DistanceTraveled { get; private set; }
        public int CoinsCollected { get; private set; }

        public float BestDistance { get; private set; }
        public int BestCoinsSingleRun { get; private set; }

        private void Awake()
        {
            if (Instance != null && Instance != this)
            {
                Destroy(gameObject);
                return;
            }
            Instance = this;
            DontDestroyOnLoad(gameObject);

            // Load bests
            BestDistance = PlayerPrefs.GetFloat("BEST_DISTANCE", 0f);
            BestCoinsSingleRun = PlayerPrefs.GetInt("BEST_COINS", 0);
        }

        private void Start()
        {
            EnterMenu();
        }

        private void Update()
        {
            if (!IsPlaying) return;

            // Increase speed over time
            RunSpeed = Mathf.Min(
                RunnerConfig.MaxForwardSpeed,
                RunSpeed + RunnerConfig.ForwardAccelerationPerSecond * Time.deltaTime
            );

            // Distance accumulates with current speed
            DistanceTraveled += RunSpeed * Time.deltaTime;
        }

        public void EnterMenu()
        {
            state = GameState.Menu;
            Time.timeScale = 1f;
        }

        public void StartGame()
        {
            ResetRunStats();
            levelGenerator.ResetAndPrewarm();
            state = GameState.Playing;
            GameStarted?.Invoke();
        }

        public void PauseGame()
        {
            if (!IsPlaying) return;
            state = GameState.Paused;
            Time.timeScale = 0f;
            GamePaused?.Invoke();
        }

        public void ResumeGame()
        {
            if (state != GameState.Paused) return;
            state = GameState.Playing;
            Time.timeScale = 1f;
            GameResumed?.Invoke();
        }

        public void TriggerGameOver()
        {
            if (state == GameState.GameOver) return;
            state = GameState.GameOver;
            Time.timeScale = 1f;

            // Save bests
            if (DistanceTraveled > BestDistance)
            {
                BestDistance = DistanceTraveled;
                PlayerPrefs.SetFloat("BEST_DISTANCE", BestDistance);
            }

            if (CoinsCollected > BestCoinsSingleRun)
            {
                BestCoinsSingleRun = CoinsCollected;
                PlayerPrefs.SetInt("BEST_COINS", BestCoinsSingleRun);
            }
            PlayerPrefs.Save();

            GameOverEvent?.Invoke();
        }

        public void AddCoin(int amount = 1)
        {
            if (!IsPlaying) return;
            CoinsCollected += Mathf.Max(1, amount);
        }

        private void ResetRunStats()
        {
            RunSpeed = RunnerConfig.InitialForwardSpeed;
            DistanceTraveled = 0f;
            CoinsCollected = 0;
        }
    }
}
```

---

## SwipeInput.cs

```csharp
using System;
using UnityEngine;

namespace EndlessRunner
{
    public class SwipeInput : MonoBehaviour
    {
        public static SwipeInput Instance { get; private set; }

        public event Action SwipedLeft;
        public event Action SwipedRight;
        public event Action SwipedUp;
        public event Action SwipedDown;

        private Vector2 startPos;
        private float startTime;
        private bool tracking;

        private void Awake()
        {
            if (Instance != null && Instance != this)
            {
                Destroy(gameObject);
                return;
            }
            Instance = this;
            DontDestroyOnLoad(gameObject);
        }

        private void Update()
        {
            if (!GameManager.Instance.IsPlaying)
            {
                // Still allow keyboard for menu testing if desired
            }

            // Touch input
            if (Input.touchCount > 0)
            {
                Touch t = Input.GetTouch(0);
                if (t.phase == TouchPhase.Began)
                {
                    tracking = true;
                    startPos = t.position;
                    startTime = Time.time;
                }
                else if (t.phase == TouchPhase.Ended && tracking)
                {
                    EvaluateSwipe(t.position, Time.time - startTime);
                    tracking = false;
                }
                else if (t.phase == TouchPhase.Canceled)
                {
                    tracking = false;
                }
            }

            // Keyboard (Editor/Desktop)
            if (Input.GetKeyDown(KeyCode.LeftArrow) || Input.GetKeyDown(KeyCode.A)) SwipedLeft?.Invoke();
            if (Input.GetKeyDown(KeyCode.RightArrow) || Input.GetKeyDown(KeyCode.D)) SwipedRight?.Invoke();
            if (Input.GetKeyDown(KeyCode.UpArrow) || Input.GetKeyDown(KeyCode.W) || Input.GetKeyDown(KeyCode.Space)) SwipedUp?.Invoke();
            if (Input.GetKeyDown(KeyCode.DownArrow) || Input.GetKeyDown(KeyCode.S)) SwipedDown?.Invoke();
        }

        private void EvaluateSwipe(Vector2 endPos, float duration)
        {
            Vector2 delta = endPos - startPos;
            if (duration > RunnerConfig.SwipeMaxTimeSeconds) return;
            if (delta.magnitude < RunnerConfig.SwipeMinDistancePixels) return;

            if (Mathf.Abs(delta.x) > Mathf.Abs(delta.y))
            {
                if (delta.x > 0) SwipedRight?.Invoke(); else SwipedLeft?.Invoke();
            }
            else
            {
                if (delta.y > 0) SwipedUp?.Invoke(); else SwipedDown?.Invoke();
            }
        }
    }
}
```

---

## PlayerController.cs

```csharp
using System.Collections;
using UnityEngine;

namespace EndlessRunner
{
    [RequireComponent(typeof(CharacterController))]
    public class PlayerController : MonoBehaviour
    {
        [Header("Lane Movement")]
        [SerializeField] private float laneChangeSpeed = 12f;
        [SerializeField] private int startingLaneIndex = 1; // 0=Left, 1=Center, 2=Right

        [Header("Jump/Slide")]
        [SerializeField] private float jumpHeight = 2.2f;
        [SerializeField] private float slideDuration = 0.8f;
        [SerializeField] private GameObject slideVisual; // Optional: lowers visuals only

        [Header("Shield Visual (optional)")]
        [SerializeField] private GameObject shieldVisual;

        private CharacterController characterController;
        private int currentLaneIndex;
        private int targetLaneIndex;

        private float verticalVelocity;
        private bool isSliding;

        // Cached default controller dims
        private float defaultHeight;
        private Vector3 defaultCenter;

        private bool shieldActive;
        private float shieldExpireTime;

        private void Awake()
        {
            characterController = GetComponent<CharacterController>();
            defaultHeight = characterController.height;
            defaultCenter = characterController.center;

            currentLaneIndex = Mathf.Clamp(startingLaneIndex, 0, RunnerConfig.LaneCount - 1);
            targetLaneIndex = currentLaneIndex;
        }

        private void OnEnable()
        {
            if (SwipeInput.Instance != null)
            {
                SwipeInput.Instance.SwipedLeft += OnSwipeLeft;
                SwipeInput.Instance.SwipedRight += OnSwipeRight;
                SwipeInput.Instance.SwipedUp += OnSwipeUp;
                SwipeInput.Instance.SwipedDown += OnSwipeDown;
            }
        }

        private void OnDisable()
        {
            if (SwipeInput.Instance != null)
            {
                SwipeInput.Instance.SwipedLeft -= OnSwipeLeft;
                SwipeInput.Instance.SwipedRight -= OnSwipeRight;
                SwipeInput.Instance.SwipedUp -= OnSwipeUp;
                SwipeInput.Instance.SwipedDown -= OnSwipeDown;
            }
        }

        private void Update()
        {
            if (!GameManager.Instance.IsPlaying) return;

            // Horizontal: move toward target lane X
            float targetX = (targetLaneIndex - 1) * RunnerConfig.LaneSpacing;
            float deltaX = targetX - transform.position.x;
            float moveX = Mathf.Clamp(deltaX, -laneChangeSpeed * Time.deltaTime, laneChangeSpeed * Time.deltaTime);

            // Vertical: gravity and jump
            if (characterController.isGrounded)
            {
                if (verticalVelocity < 0f)
                    verticalVelocity = -1f; // keeps grounded
            }
            verticalVelocity += RunnerConfig.Gravity * Time.deltaTime;

            Vector3 move = new Vector3(moveX, verticalVelocity * Time.deltaTime, GameManager.Instance.RunSpeed * Time.deltaTime);
            CollisionFlags flags = characterController.Move(move);

            // Landing reset
            if ((flags & CollisionFlags.Below) != 0 && verticalVelocity < 0f)
            {
                verticalVelocity = -1f;
            }

            // Shield timeout
            if (shieldActive && Time.time >= shieldExpireTime)
            {
                SetShield(false);
            }
        }

        private void OnSwipeLeft()
        {
            if (!GameManager.Instance.IsPlaying) return;
            targetLaneIndex = Mathf.Max(0, targetLaneIndex - 1);
        }

        private void OnSwipeRight()
        {
            if (!GameManager.Instance.IsPlaying) return;
            targetLaneIndex = Mathf.Min(RunnerConfig.LaneCount - 1, targetLaneIndex + 1);
        }

        private void OnSwipeUp()
        {
            if (!GameManager.Instance.IsPlaying) return;
            if (characterController.isGrounded && !isSliding)
            {
                float jumpVelocity = Mathf.Sqrt(-2f * RunnerConfig.Gravity * jumpHeight);
                verticalVelocity = jumpVelocity;
            }
        }

        private void OnSwipeDown()
        {
            if (!GameManager.Instance.IsPlaying) return;
            if (characterController.isGrounded && !isSliding)
            {
                StartCoroutine(SlideRoutine());
            }
        }

        private IEnumerator SlideRoutine()
        {
            isSliding = true;

            if (slideVisual != null)
            {
                slideVisual.SetActive(true);
            }

            characterController.height = defaultHeight * 0.5f;
            characterController.center = defaultCenter + Vector3.down * (defaultHeight * 0.25f);

            yield return new WaitForSeconds(slideDuration);

            if (slideVisual != null)
            {
                slideVisual.SetActive(false);
            }

            characterController.height = defaultHeight;
            characterController.center = defaultCenter;
            isSliding = false;
        }

        public void ActivateShield(float duration)
        {
            SetShield(true);
            shieldExpireTime = Time.time + duration;
        }

        public bool IsShieldActive => shieldActive;

        public void ConsumeShield()
        {
            if (!shieldActive) return;
            SetShield(false);
        }

        private void SetShield(bool active)
        {
            shieldActive = active;
            if (shieldVisual != null)
            {
                shieldVisual.SetActive(active);
            }
        }

        private void OnTriggerEnter(Collider other)
        {
            if (!GameManager.Instance.IsPlaying) return;

            if (other.CompareTag("Obstacle"))
            {
                if (shieldActive)
                {
                    ConsumeShield();
                }
                else
                {
                    GameManager.Instance.TriggerGameOver();
                }
            }
        }
    }
}
```

---

## ObjectPool.cs

```csharp
using System.Collections.Generic;
using UnityEngine;

namespace EndlessRunner
{
    public class PooledObject : MonoBehaviour
    {
        public GameObject OriginPrefab { get; set; }
    }

    public class ObjectPool : MonoBehaviour
    {
        public static ObjectPool Instance { get; private set; }

        private readonly Dictionary<GameObject, Queue<PooledObject>> prefabToPool = new();

        private void Awake()
        {
            if (Instance != null && Instance != this)
            {
                Destroy(gameObject);
                return;
            }
            Instance = this;
            DontDestroyOnLoad(gameObject);
        }

        public GameObject Get(GameObject prefab)
        {
            if (!prefabToPool.TryGetValue(prefab, out Queue<PooledObject> queue))
            {
                queue = new Queue<PooledObject>();
                prefabToPool[prefab] = queue;
            }

            PooledObject instance;
            if (queue.Count > 0)
            {
                instance = queue.Dequeue();
                instance.gameObject.SetActive(true);
            }
            else
            {
                GameObject go = Instantiate(prefab);
                instance = go.AddComponent<PooledObject>();
                instance.OriginPrefab = prefab;
            }
            return instance.gameObject;
        }

        public void Return(GameObject instance)
        {
            if (instance == null) return;
            PooledObject pooled = instance.GetComponent<PooledObject>();
            if (pooled == null || pooled.OriginPrefab == null)
            {
                Destroy(instance);
                return;
            }

            instance.SetActive(false);
            if (!prefabToPool.TryGetValue(pooled.OriginPrefab, out Queue<PooledObject> queue))
            {
                queue = new Queue<PooledObject>();
                prefabToPool[pooled.OriginPrefab] = queue;
            }
            queue.Enqueue(pooled);
        }
    }
}
```

---

## GroundChunk.cs

```csharp
using System.Collections.Generic;
using UnityEngine;

namespace EndlessRunner
{
    public class GroundChunk : MonoBehaviour
    {
        [Tooltip("Length of this chunk along Z (world units)")]
        public float length = 30f;

        private readonly List<GameObject> spawnedChildren = new();

        public void TrackSpawned(GameObject go)
        {
            go.transform.SetParent(transform, true);
            spawnedChildren.Add(go);
        }

        public void ClearSpawned()
        {
            for (int i = 0; i < spawnedChildren.Count; i++)
            {
                ObjectPool.Instance.Return(spawnedChildren[i]);
            }
            spawnedChildren.Clear();
        }

        private void OnDisable()
        {
            ClearSpawned();
        }
    }
}
```

---

## LevelGenerator.cs

```csharp
using System.Collections.Generic;
using UnityEngine;

namespace EndlessRunner
{
    public class LevelGenerator : MonoBehaviour
    {
        [Header("References")]
        [SerializeField] private Transform player;

        [Header("Chunk Prefabs (same length recommended)")]
        [SerializeField] private List<GroundChunk> chunkPrefabs;
        [SerializeField] private GroundChunk startChunkPrefab;

        [Header("Obstacles & Coins")]
        [SerializeField] private List<GameObject> obstaclePrefabs;
        [SerializeField] private GameObject coinPrefab;

        [Header("Prewarm")]
        [SerializeField] private int initialChunks = 6;

        private readonly Queue<GroundChunk> activeChunks = new();
        private float nextSpawnZ;

        public void ResetAndPrewarm()
        {
            // Clear existing
            while (activeChunks.Count > 0)
            {
                GroundChunk old = activeChunks.Dequeue();
                ObjectPool.Instance.Return(old.gameObject);
            }

            nextSpawnZ = 0f;

            // Spawn a start chunk without obstacles for safety
            if (startChunkPrefab != null)
            {
                SpawnChunk(startChunkPrefab, noObstacles: true);
            }

            while (activeChunks.Count < initialChunks)
            {
                SpawnChunk(GetRandomChunkPrefab(), noObstacles: false);
            }
        }

        private void Update()
        {
            if (!GameManager.Instance.IsPlaying) return;
            if (player == null) return;

            // Ensure enough chunks ahead
            while (nextSpawnZ - player.position.z < RunnerConfig.VisibleDistanceAhead)
            {
                SpawnChunk(GetRandomChunkPrefab(), noObstacles: false);
            }

            // Recycle chunks far behind the player
            if (activeChunks.Count > 0)
            {
                GroundChunk oldest = activeChunks.Peek();
                float endZ = oldest.transform.position.z + oldest.length;
                if (player.position.z - endZ > RunnerConfig.RecycleDistanceBehind)
                {
                    activeChunks.Dequeue();
                    ObjectPool.Instance.Return(oldest.gameObject);
                }
            }
        }

        private GroundChunk GetRandomChunkPrefab()
        {
            if (chunkPrefabs == null || chunkPrefabs.Count == 0) return startChunkPrefab;
            int idx = Random.Range(0, chunkPrefabs.Count);
            return chunkPrefabs[idx];
        }

        private void SpawnChunk(GroundChunk prefab, bool noObstacles)
        {
            GameObject go = ObjectPool.Instance.Get(prefab.gameObject);
            GroundChunk chunk = go.GetComponent<GroundChunk>();

            go.transform.SetPositionAndRotation(new Vector3(0f, 0f, nextSpawnZ), Quaternion.identity);
            go.SetActive(true);

            activeChunks.Enqueue(chunk);
            nextSpawnZ += chunk.length;

            if (!noObstacles)
            {
                PopulateChunk(chunk);
            }
        }

        private void PopulateChunk(GroundChunk chunk)
        {
            float startZ = chunk.transform.position.z + 5f; // slight buffer from beginning
            float endZ = chunk.transform.position.z + chunk.length - 5f;
            if (endZ <= startZ) return;

            // Obstacles
            int obstacleCount = Random.Range(1, 4);
            for (int i = 0; i < obstacleCount; i++)
            {
                float z = Random.Range(startZ, endZ);
                int lane = Random.Range(0, RunnerConfig.LaneCount);
                GameObject prefab = obstaclePrefabs[Random.Range(0, obstaclePrefabs.Count)];

                GameObject obs = ObjectPool.Instance.Get(prefab);
                obs.transform.position = new Vector3((lane - 1) * RunnerConfig.LaneSpacing, 0f, z);
                obs.transform.rotation = Quaternion.identity;
                obs.tag = "Obstacle"; // ensure tag
                obs.SetActive(true);
                chunk.TrackSpawned(obs);
            }

            // Coins (one or two rows)
            int coinRows = Random.Range(1, 3);
            for (int r = 0; r < coinRows; r++)
            {
                float rowZ = Random.Range(startZ, endZ);
                int path = Random.Range(0, RunnerConfig.LaneCount);
                int coinsInRow = Random.Range(4, 8);
                float spacing = 1.5f;
                for (int c = 0; c < coinsInRow; c++)
                {
                    GameObject coin = ObjectPool.Instance.Get(coinPrefab);
                    coin.transform.position = new Vector3((path - 1) * RunnerConfig.LaneSpacing, 1f, rowZ + c * spacing);
                    coin.transform.rotation = Quaternion.identity;
                    coin.tag = "Coin";
                    coin.SetActive(true);
                    chunk.TrackSpawned(coin);
                }
            }
        }
    }
}
```

---

## Coin.cs

```csharp
using UnityEngine;

namespace EndlessRunner
{
    public class Coin : MonoBehaviour
    {
        [SerializeField] private int value = 1;
        [SerializeField] private float rotateSpeed = 180f;

        private void Update()
        {
            transform.Rotate(Vector3.up, rotateSpeed * Time.deltaTime, Space.World);

            // Magnet behavior
            if (CoinMagnet.Active && CoinMagnet.Target != null)
            {
                float distance = Vector3.Distance(transform.position, CoinMagnet.Target.position);
                if (distance <= CoinMagnet.Radius)
                {
                    Vector3 dir = (CoinMagnet.Target.position - transform.position).normalized;
                    transform.position += dir * CoinMagnet.AttractSpeed * Time.deltaTime;
                }
            }
        }

        private void OnTriggerEnter(Collider other)
        {
            if (!other.CompareTag("Player")) return;

            GameManager.Instance.AddCoin(value);
            ObjectPool.Instance.Return(gameObject);
        }
    }
}
```

---

## CoinMagnet.cs

```csharp
using System.Collections;
using UnityEngine;

namespace EndlessRunner
{
    public class CoinMagnet : MonoBehaviour
    {
        public static bool Active { get; private set; }
        public static Transform Target { get; private set; }
        public static float Radius => RunnerConfig.MagnetRadius;
        public static float AttractSpeed => RunnerConfig.MagnetAttractSpeed;

        [SerializeField] private PlayerController player;

        public void Activate(float duration)
        {
            if (player == null) player = FindObjectOfType<PlayerController>();
            Target = player != null ? player.transform : null;
            if (Active) StopAllCoroutines();
            Active = true;
            StartCoroutine(MagnetRoutine(duration));
        }

        private IEnumerator MagnetRoutine(float duration)
        {
            float end = Time.time + duration;
            while (Time.time < end && GameManager.Instance.State != GameState.GameOver)
            {
                yield return null;
            }
            Active = false;
            Target = null;
        }
    }
}
```

---

## Powerups: MagnetPowerUp.cs and ShieldPowerUp.cs

```csharp
using UnityEngine;

namespace EndlessRunner
{
    public class MagnetPowerUp : MonoBehaviour
    {
        [SerializeField] private float duration = 6f;
        private void OnTriggerEnter(Collider other)
        {
            if (!other.CompareTag("Player") || !GameManager.Instance.IsPlaying) return;
            CoinMagnet magnet = FindObjectOfType<CoinMagnet>();
            if (magnet != null) magnet.Activate(duration);
            ObjectPool.Instance.Return(gameObject);
        }
    }
}
```

```csharp
using UnityEngine;

namespace EndlessRunner
{
    public class ShieldPowerUp : MonoBehaviour
    {
        [SerializeField] private float duration = 5f;
        private void OnTriggerEnter(Collider other)
        {
            if (!other.CompareTag("Player") || !GameManager.Instance.IsPlaying) return;
            PlayerController pc = other.GetComponent<PlayerController>();
            if (pc != null) pc.ActivateShield(duration);
            ObjectPool.Instance.Return(gameObject);
        }
    }
}
```

---

## FollowCamera.cs

```csharp
using UnityEngine;

namespace EndlessRunner
{
    public class FollowCamera : MonoBehaviour
    {
        [SerializeField] private Transform target;
        [SerializeField] private Vector3 offset = new Vector3(0f, 6f, -10f);
        [SerializeField] private float smoothTime = 0.12f;

        private Vector3 velocity;

        private void LateUpdate()
        {
            if (target == null) return;
            Vector3 desired = target.position + offset;
            transform.position = Vector3.SmoothDamp(transform.position, desired, ref velocity, smoothTime);
            transform.LookAt(target.position + Vector3.forward * 6f);
        }
    }
}
```

---

## UI: ScoreUI.cs

```csharp
using UnityEngine;
using UnityEngine.UI;

namespace EndlessRunner
{
    public class ScoreUI : MonoBehaviour
    {
        [Header("UI")]
        [SerializeField] private Text distanceText;
        [SerializeField] private Text coinsText;
        [SerializeField] private Text bestDistanceText;
        [SerializeField] private Text bestCoinsText;

        [Header("Panels")] 
        [SerializeField] private GameObject menuPanel;
        [SerializeField] private GameObject hudPanel;
        [SerializeField] private GameObject gameOverPanel;

        private void OnEnable()
        {
            GameManager.Instance.GameStarted += HandleGameStarted;
            GameManager.Instance.GamePaused += HandleGamePaused;
            GameManager.Instance.GameResumed += HandleGameResumed;
            GameManager.Instance.GameOverEvent += HandleGameOver;
        }

        private void OnDisable()
        {
            if (GameManager.Instance == null) return;
            GameManager.Instance.GameStarted -= HandleGameStarted;
            GameManager.Instance.GamePaused -= HandleGamePaused;
            GameManager.Instance.GameResumed -= HandleGameResumed;
            GameManager.Instance.GameOverEvent -= HandleGameOver;
        }

        private void Start()
        {
            UpdateBests();
            ShowMenu();
        }

        private void Update()
        {
            if (!GameManager.Instance.IsPlaying) return;
            distanceText.text = $"{GameManager.Instance.DistanceTraveled:0} m";
            coinsText.text = $"{GameManager.Instance.CoinsCollected}";
        }

        public void OnPlayButton()
        {
            GameManager.Instance.StartGame();
        }

        public void OnPauseButton()
        {
            GameManager.Instance.PauseGame();
        }

        public void OnResumeButton()
        {
            GameManager.Instance.ResumeGame();
        }

        public void OnRestartButton()
        {
            GameManager.Instance.StartGame();
            HideAllPanels();
            ShowHUD();
        }

        private void HandleGameStarted()
        {
            HideAllPanels();
            ShowHUD();
        }

        private void HandleGamePaused()
        {
            // Optional: show pause UI if you add one
        }

        private void HandleGameResumed()
        {
            // Optional
        }

        private void HandleGameOver()
        {
            UpdateBests();
            HideAllPanels();
            gameOverPanel.SetActive(true);
        }

        private void UpdateBests()
        {
            bestDistanceText.text = $"Best: {GameManager.Instance.BestDistance:0} m";
            bestCoinsText.text = $"Best Coins: {GameManager.Instance.BestCoinsSingleRun}";
        }

        private void HideAllPanels()
        {
            menuPanel.SetActive(false);
            hudPanel.SetActive(false);
            gameOverPanel.SetActive(false);
        }

        private void ShowMenu()
        {
            HideAllPanels();
            menuPanel.SetActive(true);
        }

        private void ShowHUD()
        {
            hudPanel.SetActive(true);
        }
    }
}
```

---

## Obstacle.cs

```csharp
using UnityEngine;

namespace EndlessRunner
{
    public class Obstacle : MonoBehaviour
    {
        // Optional: animate, move, etc.
        private void OnTriggerEnter(Collider other)
        {
            if (!other.CompareTag("Player")) return;

            PlayerController pc = other.GetComponent<PlayerController>();
            if (pc == null) return;

            if (pc.IsShieldActive)
            {
                pc.ConsumeShield();
                // Optionally VFX
                ObjectPool.Instance.Return(gameObject);
            }
            else
            {
                GameManager.Instance.TriggerGameOver();
            }
        }
    }
}
```

---

## SimpleHaptics.cs (optional mobile vibration)

```csharp
using UnityEngine;

namespace EndlessRunner
{
    public static class SimpleHaptics
    {
        public static void VibrateShort()
        {
#if UNITY_ANDROID && !UNITY_EDITOR
            Handheld.Vibrate();
#endif
        }
    }
}
```

---

## Usage Notes (Setup in Unity)

- Create an empty `GameObject` named `Systems` and add these components:
  - `GameManager`
  - `ObjectPool`
  - `SwipeInput`
  - `CoinMagnet`

- Player:
  - Create a `Player` with `CharacterController` and `PlayerController`. Tag it `Player`.
  - Optionally assign a `shieldVisual` child object.

- Camera:
  - Add `FollowCamera` to your main camera, assign `target` to `Player`.

- Level:
  - Create prefabs for `GroundChunk` with proper colliders and set `length` (e.g., 30).
  - Create several chunk prefabs for variety and set them in `LevelGenerator.chunkPrefabs`.
  - Create a `Level` GameObject with `LevelGenerator` and assign `player`, `startChunkPrefab`, chunk list, obstacle prefabs, and `coinPrefab`.

- Obstacles & Coins:
  - Make obstacle prefabs with trigger colliders and tag `Obstacle`.
  - Make a coin prefab with trigger collider and the `Coin` script; tag `Coin`.
  - Physics/triggers: Because `Player` uses `CharacterController`, ensure triggers fire by adding a kinematic `Rigidbody` to coin/obstacle prefabs (recommended). Do not add a `Rigidbody` to `Player` in this setup. Alternatively, add a kinematic `Rigidbody` to `Player` and handle triggers on `PlayerController` only (then remove `OnTriggerEnter` in `Coin` and `Obstacle` to avoid duplicate pickups/hits).

- Powerups (optional):
  - Create prefabs with trigger colliders and `MagnetPowerUp` or `ShieldPowerUp` and suitable visuals.

- UI:
  - Create canvas with `ScoreUI` and hook up `Text` fields and panels for menu, HUD, and game over.

- Build Settings:
  - Set platform to Android/iOS. Ensure all required scenes are added.

This collection forms a solid foundation for a polished, mobile-ready 3D endless runner. Tweak visuals, animations, and effects to your style.