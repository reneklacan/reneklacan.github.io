---
layout:     post
title:      Draw grid on the terrain in Unity
date:       2015-01-23 00:00:07
summary:
categories: unity3d games
image:      grid-800x422.png
---

Drawing grid on the terrain is used in lot of game genres – RTS, Simulation, Tower defense, etc. It can be done very easily in Unity.

Here is some very simple extensible solution with following features:

Respect terrain height
Option to have different texture in different parts (eg. to distinguish free and taken cells)
Configurable (you can resize grid in editor or real-time and set cell size)


{% highlight c# %}
using UnityEngine;
using System.Collections;
using System.Collections.Generic;

public class TerrainGrid : MonoBehaviour {
    public float cellSize = 1;
    public int gridWidth = 10;
    public int gridHeight = 10;
    public float yOffset = 0.5f;
    public Material cellMaterialValid;
    public Material cellMaterialInvalid;

    private GameObject[] _cells;
    private float[] _heights;

    void Start() {
        _cells = new GameObject[gridHeight * gridWidth];
        _heights = new float[(gridHeight + 1) * (gridWidth + 1)];

        for (int z = 0; z < gridHeight; z++) {
            for (int x = 0; x < gridWidth; x++) {
                _cells[z * gridWidth + x] = CreateChild();
            }
        }
    }

    void Update () {
        UpdateSize();
        UpdatePosition();
        UpdateHeights();
        UpdateCells();
    }

    GameObject CreateChild() {
        GameObject go = new GameObject();

        go.name = "Grid Cell";
        go.transform.parent = transform;
        go.transform.localPosition = Vector3.zero;
        go.AddComponent<MeshRenderer>();
        go.AddComponent<MeshFilter>().mesh = CreateMesh();

        return go;
    }

    void UpdateSize() {
        int newSize = gridHeight * gridWidth;
        int oldSize = _cells.Length;

        if (newSize == oldSize)
            return;

        GameObject[] oldCells = _cells;
        _cells = new GameObject[newSize];

        if (newSize < oldSize) {
            for (int i = 0; i < newSize; i++) {
                _cells[i] = oldCells[i];
            }

            for (int i = newSize; i < oldSize; i++) {
                Destroy(oldCells[i]);
            }
        }
        else if (newSize > oldSize) {
            for (int i = 0; i < oldSize; i++) {
                _cells[i] = oldCells[i];
            }

            for (int i = oldSize; i < newSize; i++) {
                _cells[i] = CreateChild();
            }
        }

        _heights = new float[(gridHeight + 1) * (gridWidth + 1)];
    }

    void UpdatePosition() {
        RaycastHit hitInfo;
        Ray ray = Camera.main.ScreenPointToRay(Input.mousePosition);
        Physics.Raycast(ray, out hitInfo, Mathf.Infinity, LayerMask.GetMask("Terrain"));
        Vector3 position = hitInfo.point;

        position.x -= hitInfo.point.x % cellSize + gridWidth * cellSize / 2;
        position.z -= hitInfo.point.z % cellSize + gridHeight * cellSize / 2;
        position.y = 0;

        transform.position = position;
    }

    void UpdateHeights() {
        RaycastHit hitInfo;
        Vector3 origin;

        for (int z = 0; z < gridHeight + 1; z++) {
            for (int x = 0; x < gridWidth + 1; x++) {
                origin = new Vector3(x * cellSize, 200, z * cellSize);
                Physics.Raycast(transform.TransformPoint(origin), Vector3.down, out hitInfo, Mathf.Infinity, LayerMask.GetMask("Terrain"));

                _heights[z * (gridWidth + 1) + x] = hitInfo.point.y;
            }
        }
    }

    void UpdateCells() {
        for (int z = 0; z < gridHeight; z++) {
            for (int x = 0; x < gridWidth; x++) {
                GameObject cell = _cells[z * gridWidth + x];
                MeshRenderer meshRenderer = cell.GetComponent<MeshRenderer>();
                MeshFilter meshFilter = cell.GetComponent<MeshFilter>();

                meshRenderer.material = IsCellValid(x, z) ? cellMaterialValid : cellMaterialInvalid;
                UpdateMesh(meshFilter.mesh, x, z);
            }
        }
    }

    bool IsCellValid(int x, int z) {
        RaycastHit hitInfo;
        Vector3 origin = new Vector3(x * cellSize + cellSize/2, 200, z * cellSize + cellSize/2);
        Physics.Raycast(transform.TransformPoint(origin), Vector3.down, out hitInfo, Mathf.Infinity, LayerMask.GetMask("Buildings"));

        return hitInfo.collider == null;
    }

    Mesh CreateMesh() {
        Mesh mesh = new Mesh();

        mesh.name = "Grid Cell";
        mesh.vertices = new Vector3[] { Vector3.zero, Vector3.zero, Vector3.zero, Vector3.zero };
        mesh.triangles = new int[] { 0, 1, 2, 2, 1, 3 };
        mesh.normals = new Vector3[] { Vector3.up, Vector3.up, Vector3.up, Vector3.up };
        mesh.uv = new Vector2[] { new Vector2(1, 1), new Vector2(1, 0), new Vector2(0, 1), new Vector2(0, 0) };

        return mesh;
    }

    void UpdateMesh(Mesh mesh, int x, int z) {
        mesh.vertices = new Vector3[] {
            MeshVertex(x, z),
            MeshVertex(x, z + 1),
            MeshVertex(x + 1, z),
            MeshVertex(x + 1, z + 1),
        };
    }

    Vector3 MeshVertex(int x, int z) {
        return new Vector3(x * cellSize, _heights[z * (gridWidth + 1) + x] + yOffset, z * cellSize);
    }
}
{% endhighlight %}

Because we want to have different kind of cells we have two options – one is using separate game object for every cell and second option is using one game object containing mesh with lot of submeshes. I chose first option because of simplicity and flexibility.

For every cell we create corresponding mesh. In example above you can see that it’s very simple square mesh from just 2 triangles (line 131). Y coordinate of each square’s vertex is calculated by doing raycast on the terrain (precalculated for every cell in UpdateHeights method every frame) and adding yOffset value. For better result you can experiment with square mesh with much more triangles (eg. 24).

To every mesh of the cell we assign material based on IsCellValid method which is doing raycast through the center of the cell and checking “Buildings” layer for some colliders, if there is any the cellMaterialInvalid is used (red in screenshot), otherwise cellMaterialValid is used (green).

Number of cells are managed in UpdateSize method which is checking for size changes and doing corresponding action – adding new or deleting cells.

In method UpdatePosition the grid is moved so it’s center is always under mouse cursor.

This is just an example and for real use you will probably have to modify position and resizing behaviour.
