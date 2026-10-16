---
layout: default
title: Portfolio | Vector Operations
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Vector Operations
C functions to perform basic vector mathematics for 2D and 3D vectors.

Vector2D:
```cpp
struct{
    float x;
    float y;
} typedef vector2_t;

vector2_t vec2_add(const vector2_t* a, const vector2_t* b){
    vector2_t out;
    out.x = a->x + b->x;
    out.y = a->y + b->y;
    return out;
}
vector2_t vec2_sub(const vector2_t* a, const vector2_t* b){
    vector2_t out;
    out.x = a->x - b->x;
    out.y = a->y - b->y;
    return out;
}
vector2_t vec2_mul(const vector2_t* a, float b){
    vector2_t out;
    out.x = a->x * b;
    out.y = a->y * b;
    return out;
}
float vec2_mag(const vector2_t* a){
    return sqrt(pow(a->x,2) + pow(a->y,2));
}
float vec2_cross(const vector2_t* a, const vector2_t* b){
    return (a->x * b->y) - (a->y * b->x);
}
float vec2_dot(const vector2_t* a, const vector2_t* b){
    return (a->x * b->x) + (a->y * b->y);
}
vector2_t vec2_norm(const vector2_t *a){
    return vec2_mul(a, (float)1 / vec2_mag(a));
}

```

Vector3D:
```cpp
struct{
    float x;
    float y;
    float z;
} typedef vector3_t;

vector3_t vec3_add(const vector3_t* a, const vector3_t* b){
    vector3_t out;
    out.x = a->x + b->x;
    out.y = a->y + b->y;
    out.z = a->z + b->z;
    return out;
}
vector3_t vec3_sub(const vector3_t* a, const vector3_t* b){
    vector3_t out;
    out.x = a->x - b->x;
    out.y = a->y - b->y;
    out.z = a->z - b->z;
    return out;
}
vector3_t vec3_mul(const vector3_t* a, float b){
    vector3_t out;
    out.x = a->x * b;
    out.y = a->y * b;
    out.z = a->z * b;
    return out;
}
float vec3_mag(const vector3_t* a){
    return sqrt(pow(a->x,2) + pow(a->y,2) + pow(a->z,2));
}
vector3_t vec3_cross(const vector3_t* a, const vector3_t* b){
    vector3_t out;
    out.x = (a->y * b->z) - (a->z * b->y);
    out.y = (a->z * b->x) - (a->x * b->z);
    out.z = (a->x * b->y) - (a->y * b->x);
    return out;
}
float vec3_dot(const vector3_t* a, const vector3_t* b){
    return (a->x * b->x) + (a->y * b->y) + (a->z * b->z);
}
vector3_t vec3_norm(const vector3_t *a){
    return vec3_mul(a, (float)1 / vec3_mag(a));
}
```