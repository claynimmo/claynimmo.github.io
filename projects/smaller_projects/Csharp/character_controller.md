---
layout: default
title: Portfolio |  Character Controller
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Character Controller
This code is a basic movement script for a 3D character in the Unity Engine, using the rigidbody. It includes jumping, and spring like ground collision to avoid jittering on uneven terrain.

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.InputSystem;

public class Movement : MonoBehaviour
{

    public PlayerInputs playerControls;

    public bool moveAble=true;

    [Header("Player")]
    public Rigidbody rbody;

    public Transform cam;

    public float speed = 10.0f;
    public float acceleration = 2.0f;

	public float gravity = 10.0f;


	public float maxVelocityChange = 10.0f;
    Vector3 targetVelocity;
    public float turnsmoothtime = 0.05f;
    float turnsmoothvelocity;

    [Header("Jumping")]
    private int jumps=2;
    public int maxjumps=2;
    public float jumpForce = 10;
    
    bool resetjumpdelay;
    float resetjumpdelayf = 0.3f;
    [HideInInspector] public bool grounded;


    [Header("Sound")]
    public AudioSource soundEffectSource;
    public AudioClip jumpSound;

    [Space(10)]
    [Header("keycodes")]
   
    //movement keycodes

    private InputAction move;
    private InputAction jump;

    [HideInInspector] public float horizontal;
    [HideInInspector] public float vertical;
    

    public GameObject body;

    private Vector2 currentInputVector;

    private Vector3[] forceVectors = new Vector3[2];//array to store the different force vectors.
    
    private bool hasInput = true;
    public LayerMask groundLayerMask;


    public float hoverHeight = 0.2f;
    public float hoverForce = 200f;
    public float hoverDamp = 5f;

    private void Awake(){
        playerControls = new PlayerInputs();
    }

    private void OnEnable(){
        move = playerControls.Player.Move;
        move.Enable();

        jump = playerControls.Player.Jump;
        jump.Enable();
        jump.performed += Jump;

    }

    private void OnDisable(){
        move.Disable();
        jump.Disable();
        jump.performed -= Jump;
    }
    // Start is called before the first frame update
    void Start()
    {
        jumps = 0;
        soundEffectSource = GetComponent<AudioSource>();
    }

    // Update is called once per frame
    void Update()
    {
        float targetAngle = 0;
        Inputs();
        float deadZone = 0.02f;
        bool horizontalPressed = horizontal > deadZone || horizontal < -deadZone;
        bool verticalPressed = vertical > deadZone || vertical < -deadZone;
        float updatedHorizontal = horizontal;
        float updatedVertical = vertical;

        if(!horizontalPressed){
            updatedHorizontal = 0;
        }
        if(!verticalPressed){
            updatedVertical = 0;
        }
        

        targetAngle = Mathf.Atan2(horizontal,vertical)*Mathf.Rad2Deg+cam.eulerAngles.y;
        float angle = Mathf.SmoothDampAngle(body.transform.eulerAngles.y,targetAngle,ref turnsmoothvelocity, turnsmoothtime);

        if(!moveAble){return;}

        if(updatedHorizontal!=0||updatedVertical!=0){
            body.transform.rotation = Quaternion.Euler(0,angle,0);
        }

    }

    void Inputs(){
        if(!hasInput || Variables.Paused){horizontal = 0; vertical = 0;return;}

        Vector2 moveDirection = move.ReadValue<Vector2>();
        currentInputVector = moveDirection;

        horizontal = currentInputVector.x;
        vertical = currentInputVector.y;
    }
    private void Jump(InputAction.CallbackContext context){
        if(!hasInput || Variables.Paused){return;}

        if(jumps >= maxjumps){return;}


        rbody.velocity = new Vector3(rbody.velocity.x,0,rbody.velocity.z);
        rbody.AddForce(new Vector3(0,jumpForce * rbody.mass,0));
       
        grounded=false;
        jumps+=1;

        soundEffectSource.Stop();
        soundEffectSource.PlayOneShot(jumpSound,SettingsData.Volume+0.1f);
    }
    
    void FixedUpdate() {
        if(!moveAble){return;}

        grounded = Physics.Raycast(transform.position, Vector3.down, hoverHeight + 0.01f, groundLayerMask);

        float combinedSpeed = speed;
        
        Vector3 cameraForward = Camera.main.transform.forward;
        cameraForward.y = 0;
        cameraForward.Normalize();

        Vector3 cameraRight = Camera.main.transform.right;


        rbody.AddForce(new Vector3 (0, (-gravity * rbody.mass), 0)); //apply gravity

        targetVelocity = cameraRight * horizontal + cameraForward * vertical;
        targetVelocity.Normalize();
        targetVelocity *= combinedSpeed;

 
        Vector3 velocity = rbody.velocity;
        Vector3 velocityChange = (targetVelocity - velocity);
        velocityChange.x = Mathf.Clamp(velocityChange.x, -maxVelocityChange, maxVelocityChange);
        velocityChange.z = Mathf.Clamp(velocityChange.z, -maxVelocityChange, maxVelocityChange);
        velocityChange.y = 0;

        Vector3 externalForces = GetExternalForces();
        rbody.AddForce((velocityChange*acceleration) + externalForces, ForceMode.Acceleration);

        // spring force to hover the player slightly above ground, since the rbody sometimes jitters when moving across perfectly connected hallways

        if (Physics.Raycast(transform.position, Vector3.down, out RaycastHit hit, hoverHeight * 2f) && grounded){
            float distance = hit.distance;
            float compression = 1f - (distance / hoverHeight);

            float upwardSpeed = Vector3.Dot(rbody.velocity, transform.up);
            float lift = (compression * hoverForce) - (upwardSpeed * hoverDamp);

            if(jumps == 0){resetJumps();}

            rbody.AddForce(transform.up * lift, ForceMode.Acceleration);
        }
    }

    void OnTriggerEnter(Collider other){
        if(!other.CompareTag("Player")&&!other.CompareTag("IgnoreTrigger")){
            resetJumps();
        }
    }

    public void resetJumps(){
        if(!resetjumpdelay){
            resetjumpdelay = true;
            Invoke("stopjumpdelay",resetjumpdelayf);
            jumps = 0;
            grounded=true;
        }
    }
    void stopjumpdelay(){
        resetjumpdelay = false;
    }


    public void StopMovement(){
        moveAble = false;
        rbody.velocity = Vector3.zero;
        rbody.isKinematic = true;
    }
    public void StartMovement(){
        moveAble = true;
        rbody.velocity = Vector3.zero;
        rbody.isKinematic = false;
    }


    private Vector3 GetExternalForces(){
        Vector3 returnVect = Vector3.zero;
        for(int i=0; i<forceVectors.Length;i++){
            returnVect += forceVectors[i];
        }
        return returnVect;
    }

    public void SetExternalForces(int index, Vector3 forceValue){
        index %= forceVectors.Length;
        forceVectors[index] = forceValue;
    }

    public void EnableInputs(bool enable){
        hasInput = enable;
    }

    public void SetJumps(int jumps){
        maxjumps = jumps;
    }
}

```