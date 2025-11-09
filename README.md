# Learn
#数据结构迷宫实验代码

#controller是第一人称控制器
using UnityEngine;

[RequireComponent(typeof(CharacterController))]
public class FirstPersonController : MonoBehaviour
{
    [Header("Movement Settings")]
    [SerializeField] private float walkSpeed = 5f;
    [SerializeField] private float sprintSpeed = 8f;
    [SerializeField] private float jumpHeight = 1.5f;
    [SerializeField] private float gravity = -15f;
    
    [Header("Mouse Look Settings")]
    [SerializeField] private float mouseSensitivity = 2f;
    [SerializeField] private float lookXLimit = 80f;
    
    [Header("Ground Check")]
    [SerializeField] private Transform groundCheck;
    [SerializeField] private float groundDistance = 0.3f;
    [SerializeField] private LayerMask groundMask;
    
    // Private variables
    private CharacterController characterController;
    private Camera playerCamera;
    private Vector3 moveDirection = Vector3.zero;
    private Vector3 velocity;
    private float rotationX = 0f;
    private bool isGrounded;
    
    void Start()
    {
        // Get required components
        characterController = GetComponent<CharacterController>();
        playerCamera = GetComponentInChildren<Camera>();
        
        // Safety checks
        if (playerCamera == null)
        {
            Debug.LogError("No Camera found as child of Player! Please add a Camera as a child.");
        }
        
        if (groundCheck == null)
        {
            Debug.LogWarning("Ground Check not assigned! Ground detection may not work properly.");
        }
        
        // Position player at maze start
        MazeGenerator maze = FindObjectOfType<MazeGenerator>();
        if (maze != null)
        {
            transform.position = maze.GetStartPosition();
            Debug.Log("Player positioned at maze start");
        }
        else
        {
            Debug.LogWarning("MazeGenerator not found in scene!");
        }
        
        // Lock cursor for gameplay
        Cursor.lockState = CursorLockMode.Locked;
        Cursor.visible = false;
    }
    
    void Update()
    {
        // Check if grounded
        CheckGroundStatus();
        
        // Handle all player inputs
        HandleMouseLook();
        HandleMovement();
        HandleJump();
        
        // Toggle cursor lock with Escape
        if (Input.GetKeyDown(KeyCode.Escape))
        {
            ToggleCursorLock();
        }
    }
    
    private void CheckGroundStatus()
    {
        if (groundCheck != null)
        {
            isGrounded = Physics.CheckSphere(groundCheck.position, groundDistance, groundMask);
        }
        else
        {
            // Fallback if no ground check assigned
            isGrounded = characterController.isGrounded;
        }
        
        // Reset falling velocity when grounded
        if (isGrounded && velocity.y < 0)
        {
            velocity.y = -2f; // Small negative value to keep grounded
        }
    }
    
    private void HandleMouseLook()
    {
        if (playerCamera == null) return;
        
        // Only look around when cursor is locked
        if (Cursor.lockState == CursorLockMode.Locked)
        {
            // Get mouse input
            float mouseX = Input.GetAxis("Mouse X") * mouseSensitivity;
            float mouseY = Input.GetAxis("Mouse Y") * mouseSensitivity;
            
            // Rotate camera up/down (pitch)
            rotationX -= mouseY;
            rotationX = Mathf.Clamp(rotationX, -lookXLimit, lookXLimit);
            playerCamera.transform.localRotation = Quaternion.Euler(rotationX, 0f, 0f);
            
            // Rotate player body left/right (yaw)
            transform.Rotate(Vector3.up * mouseX);
        }
    }
    
    private void HandleMovement()
    {
        // Get input from WASD or arrow keys
        float horizontal = Input.GetAxis("Horizontal"); // A/D or Left/Right
        float vertical = Input.GetAxis("Vertical");     // W/S or Up/Down
        
        // Calculate movement direction relative to player rotation
        Vector3 forward = transform.forward;
        Vector3 right = transform.right;
        
        // Remove vertical component to prevent flying
        forward.y = 0f;
        right.y = 0f;
        forward.Normalize();
        right.Normalize();
        
        moveDirection = (forward * vertical + right * horizontal).normalized;
        
        // Determine speed (sprint or walk)
        float currentSpeed = Input.GetKey(KeyCode.LeftShift) ? sprintSpeed : walkSpeed;
        
        // Apply movement
        Vector3 move = moveDirection * currentSpeed;
        characterController.Move(move * Time.deltaTime);
        
        // Apply gravity
        velocity.y += gravity * Time.deltaTime;
        characterController.Move(velocity * Time.deltaTime);
    }
    
    private void HandleJump()
    {
        // Jump when Space is pressed and player is grounded
        if (Input.GetButtonDown("Jump") && isGrounded)
        {
            // Calculate jump velocity using physics formula: v = sqrt(2 * h * g)
            velocity.y = Mathf.Sqrt(jumpHeight * -2f * gravity);
        }
    }
    
    private void ToggleCursorLock()
    {
        if (Cursor.lockState == CursorLockMode.Locked)
        {
            Cursor.lockState = CursorLockMode.None;
            Cursor.visible = true;
        }
        else
        {
            Cursor.lockState = CursorLockMode.Locked;
            Cursor.visible = false;
        }
    }
    
    // Detect when player reaches the exit
    void OnTriggerEnter(Collider other)
    {
        if (other.gameObject.name == "Exit_Gate")
        {
            Debug.Log("🎉 YOU WIN! You reached the exit!");
            OnReachExit();
        }
    }
    
   private void OnReachExit()
{
    Debug.Log("🎉 YOU WIN! You reached the exit!");
    // 解锁鼠标，显示UI
    Cursor.lockState = CursorLockMode.None;
    Cursor.visible = true;
    // 调用UI控制器，重新显示主界面
    FindObjectOfType<UIMainController>().ShowMainUIAfterWin();
}
    
    // Optional: Draw ground check sphere in editor for debugging
    void OnDrawGizmosSelected()
    {
        if (groundCheck != null)
        {
            Gizmos.color = isGrounded ? Color.green : Color.red;
            Gizmos.DrawWireSphere(groundCheck.position, groundDistance);
        }
    }
}

#MazeGen为迷宫生成代码文件
using System.Collections.Generic;
using UnityEngine;

public class MazeGenerator : MonoBehaviour
{
    [Header("Maze Settings")]
    [SerializeField] private int width = 15;
    [SerializeField] private int height = 15;
    [SerializeField] private float cellSize = 4f;
    [SerializeField] private float wallHeight = 3f;
    
    [Header("Prefabs")]
    [SerializeField] private GameObject wallPrefab;
    [SerializeField] private GameObject floorPrefab;
    [SerializeField] private GameObject exitMarkerPrefab;
    
    [Header("Generation")]
    [SerializeField] private bool generateOnStart = true;
    [SerializeField] private bool createExitOpening = true; // Remove walls around exit
    
    private Cell[,] grid;
    private Vector3 mazeOrigin;
    
    private class Cell
    {
        public int x, z;
        public bool visited;
        public bool[] walls = { true, true, true, true }; // N, E, S, W
        
        public Cell(int x, int z)
        {
            this.x = x;
            this.z = z;
            this.visited = false;
        }
    }
    
    void Start()
    {
        if (generateOnStart)
        {
            GenerateMaze();
        }
    }
    
    public void GenerateMaze()
    {
        Debug.Log("Starting maze generation...");
        ClearMaze();
        InitializeGrid();
        GenerateMazeWithDFS();
        BuildMaze3D();
        PlaceExitMarker();
        Debug.Log("Maze generation complete!");
    }
    
    private void ClearMaze()
    {
        foreach (Transform child in transform)
        {
            Destroy(child.gameObject);
        }
    }
    
    private void InitializeGrid()
    {
        grid = new Cell[width, height];
        mazeOrigin = transform.position;
        
        for (int x = 0; x < width; x++)
        {
            for (int z = 0; z < height; z++)
            {
                grid[x, z] = new Cell(x, z);
            }
        }
    }
    
    private void GenerateMazeWithDFS()
    {
        Stack<Cell> stack = new Stack<Cell>();
        Cell current = grid[0, 0];
        current.visited = true;
        stack.Push(current);
        
        while (stack.Count > 0)
        {
            current = stack.Pop();
            List<Cell> unvisitedNeighbors = GetUnvisitedNeighbors(current);
            
            if (unvisitedNeighbors.Count > 0)
            {
                stack.Push(current);
                
                Cell next = unvisitedNeighbors[Random.Range(0, unvisitedNeighbors.Count)];
                RemoveWallBetween(current, next);
                
                next.visited = true;
                stack.Push(next);
            }
        }
    }
    
    private List<Cell> GetUnvisitedNeighbors(Cell cell)
    {
        List<Cell> neighbors = new List<Cell>();
        
        // North
        if (cell.z + 1 < height && !grid[cell.x, cell.z + 1].visited)
            neighbors.Add(grid[cell.x, cell.z + 1]);
        
        // East
        if (cell.x + 1 < width && !grid[cell.x + 1, cell.z].visited)
            neighbors.Add(grid[cell.x + 1, cell.z]);
        
        // South
        if (cell.z - 1 >= 0 && !grid[cell.x, cell.z - 1].visited)
            neighbors.Add(grid[cell.x, cell.z - 1]);
        
        // West
        if (cell.x - 1 >= 0 && !grid[cell.x - 1, cell.z].visited)
            neighbors.Add(grid[cell.x - 1, cell.z]);
        
        return neighbors;
    }
    
    private void RemoveWallBetween(Cell current, Cell next)
    {
        int dx = next.x - current.x;
        int dz = next.z - current.z;
        
        if (dz == 1) // North
        {
            current.walls[0] = false;
            next.walls[2] = false;
        }
        else if (dx == 1) // East
        {
            current.walls[1] = false;
            next.walls[3] = false;
        }
        else if (dz == -1) // South
        {
            current.walls[2] = false;
            next.walls[0] = false;
        }
        else if (dx == -1) // West
        {
            current.walls[3] = false;
            next.walls[1] = false;
        }
    }
    
    private void BuildMaze3D()
    {
        // Optionally remove walls around exit to create opening
        if (createExitOpening)
        {
            Cell exitCell = grid[width - 1, height - 1];
            // Only remove north wall - keep east wall intact!
            exitCell.walls[0] = false; // North (where the green gate will be)
            // exitCell.walls[1] = false; // East - REMOVED THIS LINE
        }
        
        for (int x = 0; x < width; x++)
        {
            for (int z = 0; z < height; z++)
            {
                Cell cell = grid[x, z];
                Vector3 cellPos = mazeOrigin + new Vector3(x * cellSize, 0, z * cellSize);
                
                // Create floor
                if (floorPrefab != null)
                {
                    GameObject floor = Instantiate(floorPrefab, cellPos + new Vector3(0, 0f, 0), Quaternion.identity, transform);
                    floor.transform.localScale = new Vector3(cellSize, 0.2f, cellSize);
                    floor.name = $"Floor_{x}_{z}";
                }
                
                // Create walls
                if (wallPrefab != null)
                {
                    // North wall
                    if (cell.walls[0])
                    {
                        CreateWall(cellPos + new Vector3(0, wallHeight / 2, cellSize / 2), 
                                   new Vector3(cellSize, wallHeight, 0.2f), 
                                   $"Wall_N_{x}_{z}");
                    }
                    
                    // East wall
                    if (cell.walls[1])
                    {
                        CreateWall(cellPos + new Vector3(cellSize / 2, wallHeight / 2, 0), 
                                   new Vector3(0.2f, wallHeight, cellSize), 
                                   $"Wall_E_{x}_{z}");
                    }
                    
                    // South wall
                    if (cell.walls[2])
                    {
                        CreateWall(cellPos + new Vector3(0, wallHeight / 2, -cellSize / 2), 
                                   new Vector3(cellSize, wallHeight, 0.2f), 
                                   $"Wall_S_{x}_{z}");
                    }
                    
                    // West wall
                    if (cell.walls[3])
                    {
                        CreateWall(cellPos + new Vector3(-cellSize / 2, wallHeight / 2, 0), 
                                   new Vector3(0.2f, wallHeight, cellSize), 
                                   $"Wall_W_{x}_{z}");
                    }
                }
            }
        }
    }
    
    private void CreateWall(Vector3 position, Vector3 scale, string name)
    {
        GameObject wall = Instantiate(wallPrefab, position, Quaternion.identity, transform);
        wall.transform.localScale = scale;
        wall.name = name;
    }
    
    private void PlaceExitMarker()
    {
        if (exitMarkerPrefab != null)
        {
            // Position at the exit opening (North wall of exit cell)
            Vector3 exitPos = mazeOrigin + new Vector3((width - 1) * cellSize, wallHeight / 2, (height - 1) * cellSize + cellSize / 2);
            
            // Create a thin vertical panel as the gate (facing inward)
            GameObject exit = Instantiate(exitMarkerPrefab, exitPos, Quaternion.identity, transform);
            exit.transform.localScale = new Vector3(cellSize * 0.95f, wallHeight * 0.95f, 0.05f); // Very thin panel
            exit.name = "Exit_Gate";
            
            // Remove ALL colliders so player can walk through
            Collider[] colliders = exit.GetComponents<Collider>();
            foreach (Collider col in colliders)
            {
                Destroy(col);
            }
            
            // Force apply semi-transparent glowing material
            Renderer exitRenderer = exit.GetComponent<Renderer>();
            if (exitRenderer != null)
            {
                // Create new material instance to avoid changing prefab material
                Material gateMaterial = new Material(Shader.Find("Standard"));
                
                // Set transparent rendering mode
                gateMaterial.SetFloat("_Mode", 3); // Transparent
                gateMaterial.SetInt("_SrcBlend", (int)UnityEngine.Rendering.BlendMode.SrcAlpha);
                gateMaterial.SetInt("_DstBlend", (int)UnityEngine.Rendering.BlendMode.OneMinusSrcAlpha);
                gateMaterial.SetInt("_ZWrite", 0);
                gateMaterial.DisableKeyword("_ALPHATEST_ON");
                gateMaterial.EnableKeyword("_ALPHABLEND_ON");
                gateMaterial.DisableKeyword("_ALPHAPREMULTIPLY_ON");
                gateMaterial.renderQueue = 3000;
                
                // Set color to semi-transparent green
                Color gateColor = new Color(0, 1, 0, 0.3f); // Green with 30% opacity
                gateMaterial.color = gateColor;
                
                // Add emission for glow
                gateMaterial.EnableKeyword("_EMISSION");
                gateMaterial.SetColor("_EmissionColor", Color.green * 1.5f);
                
                exitRenderer.material = gateMaterial;
            }
            
            Debug.Log("Exit gate placed at: " + exitPos + " - Walk through to exit!");
        }
    }
    
    // Public helper methods for other systems
    public Vector3 GetStartPosition()
    {
        return mazeOrigin + new Vector3(0, 1f, 0);
    }
    
    public Vector3 GetExitPosition()
    {
        return mazeOrigin + new Vector3((width - 1) * cellSize, 1f, (height - 1) * cellSize);
    }
    
    public float GetCellSize()
    {
        return cellSize;
    }
    
    public int GetWidth()
    {
        return width;
    }
    
    public int GetHeight()
    {
        return height;
    }
    
    public bool[,] GetWalkableGrid()
    {
        bool[,] walkable = new bool[width, height];
        for (int x = 0; x < width; x++)
        {
            for (int z = 0; z < height; z++)
            {
                walkable[x, z] = true;
            }
        }
        return walkable;
    }
}
#PathfindingArrows路径生成代码
using System.Collections.Generic;
using UnityEngine;

public class PathfindArrows : MonoBehaviour
{
    [Header("Arrow Settings")]
    [SerializeField] private GameObject arrowPrefab;
    [SerializeField] private float arrowHeight = 2f;
    [SerializeField] private float arrowSpacing = 2f;
    [SerializeField] private int maxArrowsToShow = 10;
    [SerializeField] private bool showFullPath = false;
    
    [Header("Visual Settings")]
    [SerializeField] private Material pathArrowMaterial;
    [SerializeField] private Color startColor = Color.green;
    [SerializeField] private Color endColor = Color.red;
    [SerializeField] private float arrowPulseSpeed = 2f;
    
    private MazeGenerator maze;
    private List<GameObject> activeArrows = new List<GameObject>();
    private Vector2Int playerGridPos;
    private Vector2Int targetGridPos;
    private List<Vector2Int> currentPath = new List<Vector2Int>();
    
    void Start()
    {
        maze = FindObjectOfType<MazeGenerator>();
        if (maze == null)
        {
            Debug.LogError("MazeGenerator not found in scene!");
            return;
        }
        
        if (arrowPrefab == null)
        {
            Debug.LogError("Arrow prefab not assigned!");
            return;
        }
        
        // Set target to the exit position
        targetGridPos = new Vector2Int(maze.GetWidth() - 1, maze.GetHeight() - 1);
        
        Debug.Log("PathfindArrows initialized. Target: " + targetGridPos);
    }
    
    void Update()
    {
        if (maze == null) return;
        
        // Update player grid position
        Vector3 playerWorldPos = transform.position;
        Vector3 mazeOrigin = maze.transform.position;
        float cellSize = maze.GetCellSize();
        
        playerGridPos = new Vector2Int(
            Mathf.RoundToInt((playerWorldPos.x - mazeOrigin.x) / cellSize),
            Mathf.RoundToInt((playerWorldPos.z - mazeOrigin.z) / cellSize)
        );
        
        // Clamp player position to maze bounds
        playerGridPos.x = Mathf.Clamp(playerGridPos.x, 0, maze.GetWidth() - 1);
        playerGridPos.y = Mathf.Clamp(playerGridPos.y, 0, maze.GetHeight() - 1);
        
        // Update path and arrows
        UpdatePathAndArrows();
    }
    
    private void UpdatePathAndArrows()
    {
        // Clear old arrows
        ClearArrows();
        
        // Calculate new path using BFS
        currentPath = FindPathBFS(playerGridPos, targetGridPos);
        
        if (currentPath == null || currentPath.Count <= 1)
        {
            // No path found or already at target
            return;
        }
        
        // Create arrows along the path
        CreatePathArrows(currentPath);
    }
    
    private List<Vector2Int> FindPathBFS(Vector2Int start, Vector2Int target)
    {
        if (start == target) return new List<Vector2Int> { start };
        
        int width = maze.GetWidth();
        int height = maze.GetHeight();
        
        // BFS data structures
        Queue<Vector2Int> queue = new Queue<Vector2Int>();
        Dictionary<Vector2Int, Vector2Int> cameFrom = new Dictionary<Vector2Int, Vector2Int>();
        HashSet<Vector2Int> visited = new HashSet<Vector2Int>();
        
        queue.Enqueue(start);
        visited.Add(start);
        cameFrom[start] = start;
        
        // BFS algorithm
        while (queue.Count > 0)
        {
            Vector2Int current = queue.Dequeue();
            
            // Check if we reached the target
            if (current == target)
            {
                return ReconstructPath(cameFrom, start, target);
            }
            
            // Get all valid neighbors
            List<Vector2Int> neighbors = GetWalkableNeighbors(current);
            
            foreach (Vector2Int neighbor in neighbors)
            {
                if (!visited.Contains(neighbor))
                {
                    visited.Add(neighbor);
                    cameFrom[neighbor] = current;
                    queue.Enqueue(neighbor);
                }
            }
        }
        
        // No path found
        Debug.LogWarning("No path found from " + start + " to " + target);
        return null;
    }
    
    private List<Vector2Int> GetWalkableNeighbors(Vector2Int cell)
    {
        List<Vector2Int> neighbors = new List<Vector2Int>();
        int x = cell.x;
        int y = cell.y;
        
        // Check all four directions
        // North
        if (y + 1 < maze.GetHeight() && CanMoveTo(cell, new Vector2Int(x, y + 1)))
            neighbors.Add(new Vector2Int(x, y + 1));
        
        // East
        if (x + 1 < maze.GetWidth() && CanMoveTo(cell, new Vector2Int(x + 1, y)))
            neighbors.Add(new Vector2Int(x + 1, y));
        
        // South
        if (y - 1 >= 0 && CanMoveTo(cell, new Vector2Int(x, y - 1)))
            neighbors.Add(new Vector2Int(x, y - 1));
        
        // West
        if (x - 1 >= 0 && CanMoveTo(cell, new Vector2Int(x - 1, y)))
            neighbors.Add(new Vector2Int(x - 1, y));
        
        return neighbors;
    }
    
    private bool CanMoveTo(Vector2Int from, Vector2Int to)
    {
        // This is a simplified version - you might want to check actual wall data
        // For now, we'll assume all cells are walkable (no walls between them)
        // In a real implementation, you'd check the maze's wall data
        
        return true; // Temporary - always allow movement
    }
    
    private List<Vector2Int> ReconstructPath(Dictionary<Vector2Int, Vector2Int> cameFrom, Vector2Int start, Vector2Int target)
    {
        List<Vector2Int> path = new List<Vector2Int>();
        Vector2Int current = target;
        
        while (current != start)
        {
            path.Add(current);
            current = cameFrom[current];
        }
        
        path.Add(start);
        path.Reverse();
        
        return path;
    }
    
    private void CreatePathArrows(List<Vector2Int> path)
    {
        if (path == null || path.Count < 2) return;
        
        int arrowsToCreate = showFullPath ? path.Count - 1 : Mathf.Min(maxArrowsToShow, path.Count - 1);
        
        for (int i = 0; i < arrowsToCreate; i++)
        {
            Vector2Int fromCell = path[i];
            Vector2Int toCell = path[i + 1];
            
            CreateArrowBetweenCells(fromCell, toCell, i, arrowsToCreate);
        }
    }
    
    private void CreateArrowBetweenCells(Vector2Int fromCell, Vector2Int toCell, int arrowIndex, int totalArrows)
    {
        Vector3 fromWorldPos = GridToWorldPosition(fromCell);
        Vector3 toWorldPos = GridToWorldPosition(toCell);
        
        // Calculate arrow position (midpoint between cells)
        Vector3 arrowPos = (fromWorldPos + toWorldPos) * 0.5f;
        arrowPos.y = arrowHeight;
        
        // Calculate arrow rotation to point towards next cell
        Vector3 direction = (toWorldPos - fromWorldPos).normalized;
        Quaternion arrowRotation = Quaternion.LookRotation(direction, Vector3.up);
        
        // Instantiate arrow
        GameObject arrow = Instantiate(arrowPrefab, arrowPos, arrowRotation, transform);
        arrow.name = $"PathArrow_{arrowIndex}";
        
        // Apply visual settings
        ApplyArrowVisuals(arrow, (float)arrowIndex / totalArrows);
        
        activeArrows.Add(arrow);
    }
    
    private Vector3 GridToWorldPosition(Vector2Int gridPos)
    {
        Vector3 mazeOrigin = maze.transform.position;
        float cellSize = maze.GetCellSize();
        
        return mazeOrigin + new Vector3(
            gridPos.x * cellSize + cellSize * 0.5f,
            0,
            gridPos.y * cellSize + cellSize * 0.5f
        );
    }
    
    private void ApplyArrowVisuals(GameObject arrow, float progress)
    {
        Renderer renderer = arrow.GetComponent<Renderer>();
        if (renderer != null && pathArrowMaterial != null)
        {
            // Create material instance
            Material arrowMaterial = new Material(pathArrowMaterial);
            
            // Interpolate color based on progress
            Color arrowColor = Color.Lerp(startColor, endColor, progress);
            arrowMaterial.color = arrowColor;
            
            // Add emission for glow effect
            arrowMaterial.EnableKeyword("_EMISSION");
            arrowMaterial.SetColor("_EmissionColor", arrowColor * 0.5f);
            
            renderer.material = arrowMaterial;
        }
        
        // Add pulsing animation
        StartCoroutine(ArrowPulseAnimation(arrow));
    }
    
    private System.Collections.IEnumerator ArrowPulseAnimation(GameObject arrow)
    {
        Vector3 originalScale = arrow.transform.localScale;
        float pulseIntensity = 0.1f;
        
        while (arrow != null)
        {
            float pulse = Mathf.PingPong(Time.time * arrowPulseSpeed, 1f);
            float scaleMultiplier = 1f + pulse * pulseIntensity;
            arrow.transform.localScale = originalScale * scaleMultiplier;
            
            yield return null;
        }
    }
    
    private void ClearArrows()
    {
        foreach (GameObject arrow in activeArrows)
        {
            if (arrow != null)
            {
                Destroy(arrow);
            }
        }
        activeArrows.Clear();
    }
    
    // Public methods for external control
    public void SetTarget(Vector2Int newTarget)
    {
        targetGridPos = newTarget;
        UpdatePathAndArrows();
    }
    
    public void SetTargetWorldPosition(Vector3 worldPosition)
    {
        Vector3 mazeOrigin = maze.transform.position;
        float cellSize = maze.GetCellSize();
        
        Vector2Int gridPos = new Vector2Int(
            Mathf.RoundToInt((worldPosition.x - mazeOrigin.x) / cellSize),
            Mathf.RoundToInt((worldPosition.z - mazeOrigin.z) / cellSize)
        );
        
        SetTarget(gridPos);
    }
    
    public void TogglePathVisibility(bool visible)
    {
        foreach (GameObject arrow in activeArrows)
        {
            if (arrow != null)
            {
                arrow.SetActive(visible);
            }
        }
    }
    
    public List<Vector2Int> GetCurrentPath()
    {
        return new List<Vector2Int>(currentPath);
    }
    
    public Vector2Int GetPlayerGridPosition()
    {
        return playerGridPos;
    }
    
    public Vector2Int GetTargetGridPosition()
    {
        return targetGridPos;
    }
    
    void OnDestroy()
    {
        ClearArrows();
    }
}
#UImanager UI界面设计代码
using UnityEngine;
using UnityEngine.UI;

public class UIMainController : MonoBehaviour
{
    [Header("UI引用")]
    [SerializeField] private Button startBtn;
    [SerializeField] private Button quitBtn;
    [SerializeField] private GameObject mainUI; // 就是我们创建的MainUI面板

    [Header("游戏核心引用")]
    [SerializeField] private MazeGenerator mazeGenerator; // 拖拽场景中的迷宫生成器对象
    [SerializeField] private FirstPersonController playerController; // 拖拽场景中的玩家对象

    void Start()
    {
        // 绑定按钮点击事件
        startBtn.onClick.AddListener(OnStartGame);
        quitBtn.onClick.AddListener(OnQuitGame);

        // 初始状态：显示UI，锁定玩家输入（避免未开始就移动）
        mainUI.SetActive(true);
        if (playerController != null)
        {
            playerController.enabled = false; // 禁用玩家控制
        }
    }

    // 开始游戏逻辑
    private void OnStartGame()
    {
        // 隐藏UI面板
        mainUI.SetActive(false);

        // 生成新迷宫（确保每次开始都是新迷宫）
        if (mazeGenerator != null)
        {
            mazeGenerator.GenerateMaze();
        }

        // 启用玩家控制（允许移动、视角操作）
        if (playerController != null)
        {
            playerController.enabled = true;
            // 重新锁定鼠标（符合第一人称游戏操作）
            Cursor.lockState = CursorLockMode.Locked;
            Cursor.visible = false;
        }
    }

    // 退出游戏逻辑
    private void OnQuitGame()
    {
        // 编辑器中退出播放模式，打包后退出程序
        #if UNITY_EDITOR
        UnityEditor.EditorApplication.isPlaying = false;
        #else
        Application.Quit();
        #endif
    }

    // 通关后重新显示UI（对接玩家胜利逻辑）
    public void ShowMainUIAfterWin()
    {
        mainUI.SetActive(true);
        if (playerController != null)
        {
            playerController.enabled = false; // 禁用玩家控制
            Cursor.lockState = CursorLockMode.None;
            Cursor.visible = true;
        }
    }
}
