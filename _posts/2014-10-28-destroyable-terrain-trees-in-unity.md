---
layout:     post
title:      Destroyable terrain trees in Unity
date:       2014-10-28 00:00:07
summary:
categories: unity3d games
image:      trees-800x413.png
---

To achieve destroyable terrain trees there is a little trick. You have to use trees without colliders and then create colliders dynamically from editor or at the start of runtime.

Following solution creates object with collider for every tree at the start of runtime. Every object collider has a index reference to the treeInstances array from TerrainData so when you his this object’s collider you are able to locate terrain tree instance.

Created collider has same size of every kind of tree but this can be customized and branched based on tree prefab that can be accessed through terrain.treePrototypes[treeInstance.prototypeIndex].prefab.

Attach TreeTerrain.cs script to your terrain.

{% highlight c# %}
public class TreeTerrain : MonoBehaviour {
    public Terrain terrain;
    private TreeInstance[] _originalTrees;

    void Start () {
        terrain = GetComponent<Terrain>();

        // backup original terrain trees
        _originalTrees = terrain.terrainData.treeInstances;

        // create capsule collider for every terrain tree
        for (int i = 0; i < terrain.terrainData.treeInstances.Length; i++) {
            TreeInstance treeInstance = terrain.terrainData.treeInstances[i];
            GameObject capsule = GameObject.CreatePrimitive(PrimitiveType.Capsule);

            CapsuleCollider capsuleCollider = capsule.collider as CapsuleCollider;
            capsuleCollider.center = new Vector3(0, 5, 0);
            capsuleCollider.height = 10;

            DestroyableTree tree = capsule.AddComponent<DestroyableTree>();
            tree.terrainIndex = i;

            capsule.transform.position = Vector3.Scale(treeInstance.position, terrain.terrainData.size);
            capsule.tag = "Tree";
            capsule.transform.parent = terrain.transform;
            capsule.renderer.enabled = false;
        }
    }

    void OnApplicationQuit() {
        // restore original trees
        terrain.terrainData.treeInstances = _originalTrees;
    }
}
{% endhighlight %}

And this script is dynamically attached to created objects with collider from TreeTerrain.cs script above.

{% highlight c# %}
public class DestroyableTree : MonoBehaviour {
    public int terrainIndex;

    public void Delete() {
        Terrain terrain = Terrain.activeTerrain;

        List<TreeInstance> trees = new List<TreeInstance>(terrain.terrainData.treeInstances);
        trees[terrainIndex] = new TreeInstance();
        terrain.terrainData.treeInstances = trees.ToArray();

        Destroy(gameObject);
    }
}
{% endhighlight %}

For example the flying fireball with trigger collider that would be able to destroy terrain tree can look like this.

{% highlight c# %}
public class Fireball : MonoBehaviour {
    void OnTriggerEnter(Collider collider) {
        DestroyableTree tree = collider.gameObject.GetComponent<DestroyableTree>();

        if (tree == null)
            return;

        tree.Delete();
    }
}
{% endhighlight %}

More flexible and general approach would be to create some interface (eg. IHittable or IDestroyable or IDamagable) with methods like Hit or Destroy. Terrain tree and other damagable objects would implement this interface and then Fireball class could look like this.

{% highlight c# %}
public class Fireball : MonoBehaviour {
    void OnTriggerEnter(Collider collider) {
        IHittable hittable = collider.gameObject.GetComponent(typeof(IHittable)) as IHittable;;

        if (hittable == null)
            return;

        tree.Hit();
    }
}
{% endhighlight %}

Using approach described in this post you don’t need to call TerrainData.SetHeights to destroy colliders from removed trees so you will avoid some performance issues.


