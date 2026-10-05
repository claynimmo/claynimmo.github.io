---
layout: default
title: Portfolio |  Enemy Controller
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Enemy Controller
This code contains the classes used to controll the enemy AI to randomly walk around the generated map from the [Room Generation](roomgeneration.md) code.

The EnemyController, EnemyPathfinding, and EnemyMovementManager are placed in the parent object of the enemy, alongside the rigidbody, collision, and animator. The EnemyNoticer script is applied to a an object dedicated to the collision triggers determining the view range. Ensure the tag or layer is set such that the trigger does not trigger unrelated scripts OnTriggerEnter functions.
```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class EnemyController : MonoBehaviour
{
    private EnemyPathfinding pathFinding;
    private EnemyMovementManager movement;

    private Transform target;
    private bool isAgroed = false;

    private List<Vector3> pendingChasePath = null;
    private bool isComputingPath = false;
    [SerializeField] private float chaseUpdateInterval = 0.4f; // update path every 0.4 seconds
    private float chaseTimer = 0f;

    void Awake(){
        pathFinding = GetComponent<EnemyPathfinding>();
        movement = GetComponent<EnemyMovementManager>();
    }

    void Update(){
        if(Variables.Paused) return;
        if(movement.IsIdle()){
            if(isAgroed){
                TryUpdatePathThreaded();
                if(pendingChasePath != null){
                    movement.FollowPath(pendingChasePath);
                    pendingChasePath = null;
                }
            }
            else{GridCell t = pathFinding.GetRandomCell(pathFinding.currentCell);
                var path = pathFinding.GetNavigationPath(pathFinding.currentCell, t);
                var worldPath = pathFinding.GetPathAsCoordinates(path);

                movement.FollowPath(worldPath);
            }
        }
    }

    private void TryUpdatePathThreaded(){
        if (isComputingPath || target == null)
            return;

        chaseTimer += Time.deltaTime;
        if (chaseTimer < chaseUpdateInterval)
            return;

        chaseTimer = 0f;
        isComputingPath = true;

        // Capture Unity data on main thread
        GridCell startCell = pathFinding.currentCell;
        GridCell playerCell = pathFinding.GetGridFromTransform(target);

        // Run A* on background thread
        System.Threading.Tasks.Task.Run(() =>{
            var path = pathFinding.GetChasePath(startCell, playerCell);
            var worldPath = pathFinding.GetPathAsCoordinates(path);

            // Store result for main thread
            pendingChasePath = worldPath;

            if(pendingChasePath.Count == 0){
                GridCell t = pathFinding.GetRandomCell(pathFinding.currentCell);
                var p= pathFinding.GetNavigationPath(pathFinding.currentCell, t);
                pendingChasePath = pathFinding.GetPathAsCoordinates(p);
            }

            isComputingPath = false;
        });
    }


    public void StartAgro(Transform t){
        target = t;
        isAgroed = true;
    }

    public void StopAgro(){
        target = null;
        isAgroed = false;
        movement.StopTarget();
    }

    public void DirectAgro(){
        movement.TargetPlayer(target);
    }

    public void StopDirectAgro(){
        movement.StopTarget();
        chaseTimer = chaseUpdateInterval;
    }
}
```

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class EnemyMovementManager : MonoBehaviour
{

    public float speed = 4f;
    [SerializeField] private float unagroedSpeed = 10f;
    private Queue<Vector3> waypoints = new();

    private bool targetPlayer;
    private Transform target;

    public float gravityForce = 30;
    public float rotSmooth = 1.05f;

    private bool grounded;

    private bool jumped;

    private Rigidbody rbody;
    private int currentJumps;
    [SerializeField] private int amountOfJumps = 1;
    [SerializeField] private float jumpForce = 10;
    [SerializeField] private float jumpCooldown = 3;
    [SerializeField] private float jumpRaycastDistance = 0.5f;
    [SerializeField] private LayerMask rayLayerMask;
    [SerializeField] private Animator anim;

    public LayerMask groundLayerMask;


    [SerializeField] private float hoverHeight = 0.7f;
    [SerializeField] private float hoverForce = 200f;
    [SerializeField] private float hoverDamp = 10f;

    private bool isWaiting = false;
    private float lookAroundTime = 4;

    void Awake(){
        rbody = GetComponent<Rigidbody>();
    }

    public void FollowPath(List<Vector3> points){
        waypoints.Clear();
        waypoints = new Queue<Vector3>(points);
    }

    void Update(){
        if(Variables.Paused) return;
        if(anim != null)
            anim.SetBool("look",isWaiting);
        if(waypoints.Count == 0 && !targetPlayer)
            return;

        Vector3 target;
        if(targetPlayer && this.target != null){
                target = this.target.position;
        }
        else{
            if(waypoints.Count == 0)
                return;
            target = waypoints.Peek();
        }

        float currentSpeed = targetPlayer ? speed : unagroedSpeed;

        Vector3 dir = (target - transform.position);
        dir = new Vector3(dir.x,0,dir.z);

        Quaternion targetRotation = Quaternion.LookRotation(dir);
        transform.rotation = Quaternion.Slerp(transform.rotation, targetRotation, Time.deltaTime * rotSmooth);
        dir = Vector3.Normalize(dir);
        Vector3 newDirection = dir*currentSpeed;
        
        RaycastHit wallHit;
        if(Physics.Raycast(transform.position, dir, out wallHit, 1f, rayLayerMask)){
            // try moving sideways instead of pushing forward, to not get stuck in walls
            Vector3 slide = Vector3.Cross(Vector3.up, wallHit.normal);
            dir = slide.normalized;
        }
        rbody.velocity = new Vector3(newDirection.x,rbody.velocity.y,newDirection.z);

        if(!jumped && currentJumps > 0){
            RaycastHit hit;
            if(Physics.Raycast(transform.position,transform.forward, out hit, jumpRaycastDistance,rayLayerMask)){
                jumped = true;
                currentJumps -= 1;
                grounded = false;
                rbody.velocity = new Vector3(rbody.velocity.x,0,rbody.velocity.z);
                rbody.AddForce(Vector3.up * rbody.mass * jumpForce);
                Invoke("ResetJumps",jumpCooldown);
            }
        }

        if (Vector3.Distance(new Vector3(transform.position.x, 0, transform.position.z), new Vector3(target.x, 0, target.z)) < 3f && !targetPlayer && !isWaiting){
            waypoints.Dequeue();
            if(waypoints.Count == 0 && !targetPlayer){
                StartCoroutine(LookAround());
            }
        }

    }

    IEnumerator LookAround(){
        isWaiting = true;
        rbody.velocity = Vector3.zero;
        yield return new WaitForSeconds(lookAroundTime);
        isWaiting = false;
    }

    void FixedUpdate(){
        rbody.AddForce(new Vector3 (0, (-gravityForce * rbody.mass), 0)); //apply gravity
        grounded = Physics.Raycast(transform.position, Vector3.down, hoverHeight + 0.01f, groundLayerMask);
        if (Physics.Raycast(transform.position, Vector3.down, out RaycastHit hit, hoverHeight * 2f) && grounded){
            float distance = hit.distance;
            float compression = 1f - (distance / hoverHeight);

            float upwardSpeed = Vector3.Dot(rbody.velocity, transform.up);
            float lift = (compression * hoverForce) - (upwardSpeed * hoverDamp);

            rbody.AddForce(transform.up * lift, ForceMode.Acceleration);
        }
    }

    void OnTriggerEnter(Collider other){
        if(other.CompareTag("IgnoreTrigger")) return;
        currentJumps = amountOfJumps;
    }
    

    void ResetJumps(){
        jumped = false;
    }

    public bool IsIdle(){
        return waypoints.Count == 0 && !isWaiting;
    }

    public void TargetPlayer(Transform target){
        this.target = target;
        targetPlayer = true;
        if(isWaiting){
            StopAllCoroutines();
            isWaiting = false;
        }
    }

    public void StopTarget(){
        target = null;
        targetPlayer = false;
        waypoints.Clear();
    }
}

```

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class EnemyNoticer : MonoBehaviour
{
    public EnemyController controller;

    private Transform target;
    private bool checkAgro = false;
    private bool directAgro = false;
    private float currentNotSeenTime = 0;

    [SerializeField] private LayerMask lookMask;

    [SerializeField] private float maxViewDistance = 50;
    [SerializeField] private float maxNotSeenTime = 50;
    void Update(){
        if(Variables.Paused) return;
        if(!checkAgro){return;}
        Vector3 dir = (target.position - transform.position).normalized;
        if(Physics.Raycast(transform.position, dir, out RaycastHit hit, maxViewDistance, lookMask,QueryTriggerInteraction.Ignore)){

            if(hit.collider.transform == target){
                controller.DirectAgro();
                currentNotSeenTime = 0;
                directAgro = true;
                return;
            }
        }
        currentNotSeenTime += Time.deltaTime;
        if(currentNotSeenTime > maxNotSeenTime){
            currentNotSeenTime = 0;
            StopAllAgro();
        }
        if(directAgro){
            controller.StopDirectAgro();
            directAgro = false;
        }
            
    }

    void OnTriggerEnter(Collider other){
        if(!other.CompareTag("Player")) return;
        if(checkAgro) return;
        Vector3 dir = (other.transform.position - transform.position).normalized;
        if(Physics.Raycast(transform.position, dir, out RaycastHit hit, maxViewDistance, lookMask,QueryTriggerInteraction.Ignore)){
            if(hit.collider.transform == other.transform){
                target = other.transform;
                checkAgro = true;
                currentNotSeenTime = 0;
                controller.StartAgro(other.transform);
            }
        }
    }

    private void StopAllAgro(){
        target = null;
        checkAgro = false;
        currentNotSeenTime = 0;
        controller.StopAgro();
    }


}
```

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class EnemyPathfinding : MonoBehaviour
{
    [SerializeField] private MapGenerator map;

    private Grid<GridCell> grid;
    private int gridWidth;
    private int gridHeight;
    private int cellSize;

    private AreaType startArea = new();

    public GridCell currentCell;

    public Vector3 positionOffset = new Vector3(0,2,0);

    private int consecutiveNoFinding = 0;
    private int maxConsecutiveNoFinding = 10;

    IEnumerator Wait(){
        yield return new WaitForSeconds(10);
        Initialize(map);
    }

    void Awake(){
        StartCoroutine(Wait());
    }


    public void Initialize(MapGenerator map){
        var (width, height, cellSize, grid) = map.GetGrid();
        gridWidth = width;
        gridHeight = height;
        this.cellSize = cellSize;
        this.grid = grid;
        currentCell = GetRandomCell(null);
        startArea = currentCell.areaType;
        transform.position = grid.GetWorldCenterPosition(currentCell.x, currentCell.y) + positionOffset;
    }

    public void MoveToRandomPoint(){
        GridCell cell = GetRandomCell(currentCell);
        transform.position = grid.GetWorldCenterPosition(cell.x, cell.y) + positionOffset;
    }

    void Update(){
        if(grid == null){return;}
        grid.GetXY(transform.position, out int x, out int y);
        currentCell = grid.GetValue(x, y);
    }

    public GridCell GetGridFromTransform(Transform t){
        grid.GetXY(t.position, out int x, out int y);
        return grid.GetValue(x,y);
    }

    public GridCell GetRandomCell(GridCell? currentCell){
        GridCell? chosen = null;
        int count = 0;
        for(int x = 0; x < gridWidth; x++){
            for(int y = 0; y < gridHeight; y++){
                GridCell cell = grid.GetValue(x,y);
                if(cell == null || cell == currentCell || !cell.isHallway)
                    continue;
                
                count ++;
                if(Random.Range(0, count) == 0)
                    chosen = cell;
            }
        }
        if(chosen == null)
            return currentCell;
        return chosen!;
    }

    public List<GridCell> GetNavigationPath(GridCell startCell, GridCell endCell){
        if(grid == null) return new();
        var path = AStar.FindPath(grid, startCell, endCell, NavigateCost);
        if(path.Count == 0){
            consecutiveNoFinding ++;
            if(consecutiveNoFinding >= maxConsecutiveNoFinding){
                MoveToRandomPoint();
                path = AStar.FindPath(grid, startCell, endCell, NavigateCost);
            }
        }
        else
            consecutiveNoFinding = 0;
        return path;
    }

    public List<GridCell> GetChasePath(GridCell startCell, GridCell endCell){
        if(grid == null) return new();
        return AStar.FindPath(grid, startCell, endCell, ChaseCost);
    }

    public List<Vector3> GetPathAsCoordinates(List<GridCell> path){
        List<Vector3> p = new();
        foreach(GridCell c in path){
            p.Add(grid.GetWorldCenterPosition(c.x,c.y));
        }
        return p;
    }

    // cost of the cell for the Astar search, to prioritize certain types of cells
    private float NavigateCost(GridCell cell){
        int hallwayCost = 1;
        int otherAreaCost = 2; // slightly priorize staying inside the same area
        int emptyCost = -1;
        int roomCost = -1;
        int defaultCost = -1;

        if(cell.isHallway)
            if(cell.areaType != startArea)
                return otherAreaCost;
            else return hallwayCost;
        else if(cell.isBlank)
            return emptyCost;
        else if(cell.used)
            return roomCost;
        return defaultCost;
    }
    private float ChaseCost(GridCell cell){
        int hallwayCost = 1;
        int otherAreaCost = 1; // make this the same when chasing, to prioritize shortest path
        int emptyCost = -1;
        int roomCost = -1;
        int defaultCost = -1;

        if(cell.isHallway)
            if(cell.areaType != startArea)
                return otherAreaCost;
            else return hallwayCost;
        else if(cell.isBlank)
            return emptyCost;
        else if(cell.used)
            return roomCost;
        return defaultCost;
    }
}
```