---
layout:     post
title:      Falling cube fracturing on a groundplane
summary:    A Omniverse USD example of a physically correct cube falling and fracturing on a groundplane
categories: omniverse
tags:
 - usd
 - omniverse
 - physx
 - physics
 - collision
 - blast
 - rigidbody
---

Let's create a physical simulation of a cube mesh falling under the effect of gravity on a groundplane and fracturing itself during impact in Omniverse Create/Composer.

![Thumper](/assets/images/cube_falling_fracturing_on_groundplane.gif)

Most of the steps above can be accomplished via UI, anyway we'll use USD Python and Kit APIs to set it up. The following code can be run directly from the Create/Composer script editor or from within a custom extension.

> Two extensions are required: omni.physx and blast destruction. You can install and enable them from the Extensions management window in Create/Composer. Always look at the console logs to find out whether something's off.

```python
# Creates a physics enabled ground plane and a mesh cube which
# will fall under the effect of gravity and fracture on impact
# on the ground plane in a physically correct way when simulation starts

from pxr import *
from carb import *
from omni.ui import *
from omni.usd import *
from omni.kit.commands import *
import asyncio
import time
from omni.blast.bindings import _blast

def reset_current_stage():
    # Resets the current stage entirely
    omni.kit.stage_templates.new_stage(template=None)

reset_current_stage()

# get current context and the new stage, assumes this script is
# being run in Omniverse Create/Composer
usd_context = omni.usd.get_context()
stage = usd_context.get_stage()

# set up stage
UsdGeom.SetStageUpAxis(stage, UsdGeom.Tokens.y)
UsdGeom.SetStageMetersPerUnit(stage, 1)

# create a physics scene for the physics simulation to run in
sceneSdfPath = omni.usd.get_stage_next_free_path(stage, "/physicsScene", True)
scene = UsdPhysics.Scene.Define(stage, sceneSdfPath)
scene.CreateGravityDirectionAttr().Set(Gf.Vec3f(0.0, -1.0, 0.0))
scene.CreateGravityMagnitudeAttr().Set(69.81)


def create_ground_plane(stage,
    planeSdfPath: Sdf.Path,
    position: Gf.Vec3f,
    sizeOfVisualizationMesh: int,
    rgbColorOfVisualizationMesh: Gf.Vec3f) -> UsdGeom.Xform:
    """
        Create a ground plane (an endless UsdGeom.Plane with CollisionAPI
        with a smaller quad mesh to help visualizing it) with the up axis Y
    """

    # create an xform to group the entire plane and to avoid nesting geom prims
    planeXformSdfPath = omni.usd.get_stage_next_free_path(
        stage, planeSdfPath, True)
    planeXform = UsdGeom.Xform.Define(stage, planeXformSdfPath)
    planeXform.AddTranslateOp().Set(position)
    planeXform.AddOrientOp().Set(Gf.Quatf(1.0))
    planeXform.AddScaleOp().Set(Gf.Vec3f(1.0))

    # Add a simple quad mesh to visualize the plane.
    geomPlaneSdfPath = planeXformSdfPath + "/planeMesh"
    halfSize = sizeOfVisualizationMesh / 2
    points = [
        Gf.Vec3f(-halfSize, 0.0, -halfSize),
        Gf.Vec3f(halfSize, 0.0, -halfSize),
        Gf.Vec3f(halfSize, 0.0, halfSize),
        Gf.Vec3f(-halfSize, 0.0, halfSize),
    ]
    normals = [Gf.Vec3f(0, 1, 0), Gf.Vec3f(0, 1, 0),
               Gf.Vec3f(0, 1, 0), Gf.Vec3f(0, 1, 0)]
    indices = [3, 2, 1, 0]
    vertexCounts = [4]
    planeMesh = UsdGeom.Mesh.Define(stage, geomPlaneSdfPath)
    # Fill in VtArrays for face vertex counts, indices, points, normals, etc.
    planeMesh.CreateFaceVertexCountsAttr().Set(vertexCounts)
    planeMesh.CreateFaceVertexIndicesAttr().Set(indices)
    planeMesh.CreatePointsAttr().Set(points)
    planeMesh.CreateDoubleSidedAttr().Set(False)
    planeMesh.CreateNormalsAttr().Set(normals)
    planeMesh.CreateDisplayColorAttr().Set([rgbColorOfVisualizationMesh])

    # Add a UsdGeom.Plane, this defines an endless infinite plane with an up axis.
    # Also add a CollisionAPI to it so there will be an infinite
    # plane acting as 'scene table'
    collisionPlanePath = planeXformSdfPath + "/collisionPlane"
    planeGeom = UsdGeom.Plane.Define(stage, collisionPlanePath)
    # do not render it unless "show guides" was requested
    planeGeom.CreatePurposeAttr().Set("guide")
    planeGeom.CreateAxisAttr().Set(UsdGeom.Tokens.y)

    UsdPhysics.CollisionAPI.Apply(planeGeom.GetPrim())

    return planeXformSdfPath


# Create a ground plane for the cube to collide against
grayColor = Gf.Vec3f(0.5)  # 0.5 0.5 0.5
originPosition = Gf.Vec3f(0.0)  # 0 0 0
groundPlaneXformSdfPath = create_ground_plane(stage, "/groundPlane",
    originPosition, 10, grayColor)


# Create a cube mesh affected by gravity thanks to the RigidBodyAPI
cubeSdfPath = omni.usd.get_stage_next_free_path(stage, "/cube", True)
def create_cube_mesh(stage,
    cubeSdfPath: Sdf.Path,
    position: Gf.Vec3f,
    rgbColorOfVisualizationMesh: Gf.Vec3f) -> Usd.Prim:
    """
        Create a cube mesh (this is different from the cube shape! This one is
        a mesh and can be fractured!) This works the same as before,
        but kit has a built-in API to do that
    """
    cubeSdfPath = omni.usd.get_stage_next_free_path(stage, cubeSdfPath, True)
    success, meshSdfPath = omni.kit.commands.execute(
        "CreateMeshPrimWithDefaultXform", prim_path=cubeSdfPath,
        prim_type="Cube", u_patches=2, v_patches=2
    )

    cubePrim = stage.GetPrimAtPath(meshSdfPath)
    #cubeMesh = UsdGeom.Mesh(cubePrim)
    #cubeMesh.CreateDisplayColorAttr().Set([rgbColorOfVisualizationMesh])

    # Apply translation to the prim, this needs to be done on the Xformable component
    # cubeXf = UsdGeom.Xformable(cubePrim)
    # Note that UsdGeom.Xformable.AddTranslateOp must not be called here:
    # the kit function _already_ adds a translateOp to the Xform
    cubePrim.GetAttribute("xformOp:translate").Set(position)

    # Assign a rigid body API to the cube, gravity and forces will now affect it
    rigidBodyAPI = UsdPhysics.RigidBodyAPI.Apply(cubePrim)
    massApi = UsdPhysics.MassAPI.Apply(cubePrim)
    # massApi.CreateMassAttr(1.0)
    # massApi.CreateDensityAttr(0.004) # never set too small a density
    # here or fracturing won't work

    # Assign a collider API to the cube, the sphere will now collide
    # against other objects
    collisionAPI = UsdPhysics.CollisionAPI.Apply(cubePrim)
    # Apply a mesh collision API that allows us to further customize
    # the physical collider type
    meshCollisionAPI = UsdPhysics.MeshCollisionAPI.Apply(cubePrim)
    # A dynamic object (with a rigid body) cannot have a triangle mesh
    # collider, use a convex hull instead
    meshCollisionAPI.CreateApproximationAttr().Set("convexHull")
    return cubePrim

redColor = Gf.Vec3f(0.9, 0, 0)
cubePrim = create_cube_mesh(stage, "/cubeMesh", Gf.Vec3d(0, 7, 0), redColor)

# Create a red material for the cube by using a simple UsdPreviewSurface shader
# (a preview material, very simple) with basic properties
mtlSdfPath = Sdf.Path("/World/Looks/PreviewSurface")
mtl = UsdShade.Material.Define(stage, mtlSdfPath)
shader = UsdShade.Shader.Define(stage, mtlSdfPath.AppendPath("Shader"))
shader.CreateIdAttr("UsdPreviewSurface")
shader.CreateInput("diffuseColor", Sdf.ValueTypeNames.Color3f).Set((1.0, 0.0, 0.0))
shader.CreateInput("roughness", Sdf.ValueTypeNames.Float).Set(0.5)
shader.CreateInput("metallic", Sdf.ValueTypeNames.Float).Set(0.0)
mtl.CreateSurfaceOutput().ConnectToSource(shader.ConnectableAPI(), "surface")

# Binding a material can be done with Kit functions or with plain USD APIs
# since this is a feature of the USD format, not Kit-specific
# with Kit APIs:
#   omni.kit.commands.execute('BindMaterial',
# 	    material_path='/World/Looks/PreviewSurface',
# 	    prim_path=['/World/cubeMesh'],
# 	    strength=['weakerThanDescendants'])
# with USD APIs:
cubeMesh = UsdGeom.Mesh(cubePrim)
UsdShade.MaterialBindingAPI(cubeMesh).Bind(mtl)

# Now use blast (you need to have the "blast destruction" extension installed
# and enabled in your Kit application) to fracture the cube mesh randomly
blastInterface = _blast.acquire_blast_interface()
omni.kit.commands.execute('BlastFracturePrims',
                          blast_interface=blastInterface,
                          prim_paths=[str(cubePrim.GetPath())],
                          blast_material=str(UsdShade.Material(cubePrim)),
                          num_voronoi_sites=32,
                          # just something extremely low
                          default_contact_threshold=9.999999747378752e-05,
                          impact_damage_scale=1.0,
                          allow_self_damage=False,
                          # just something extremely high
                          default_max_contact_impulse=12345.0,
                          interior_uv_scale=2.1)

# Optional: center the viewport default perspective camera to the ground plane
omni.kit.commands.execute('FramePrimsCommand',
                          prim_to_move=Sdf.Path('/OmniverseKit_Persp'),
                          prims_to_frame=[str(groundPlaneXformSdfPath)],
                          time_code=Usd.TimeCode(),
                          usd_context_name='',
                          aspect_ratio=1.7777777777777777)

```

After successfully setting up our scene with the code above, starting the simulation should produce a physically accurate simulation of a cube mesh falling onto a groundplane and shattering itself in fragments.