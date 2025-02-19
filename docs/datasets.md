# base_3d_dataset
Base class for all 3d dataset. Contains image/mask(optional)/camera.
Support precache_ray/norm_cam_pose/rescale_image_pose/get_item in a uniform way.
- Each dataset contains an `identifier` that is a string separating the scene from the same dataset.
(like scan_id, scene_name, etc)

## base_3d_pc_dataset
Based on `base_3d_dataset`, it provides functions mainly on point cloud adjustment.
Points are in world coordinate.
- point_cloud: a dict with 'pts'/'color'/'vis'. Last two are optional.

------------------------------------------------------------------------
# Samples
For some dataset sample, please see `configs/datasets` for samples. They can be used in unittest for visualization.

------------------------------------------------------------------------
# Some configs in data processing:
- img_scale: Resize image by this scale. `>1` means larger image.
Will change intrinsic for actual re-projection as well, but not change extrinsic.
- scale_radius: Rescale all camera pose such that cameras are roughly align on the surface of sphere with such radius.
Will not touch intrinsic. If point cloud exists, rescale them by same factor to keep consistency.
    - This actual cam radius will be adjusted by a factor of `1.05` which ensure cam `inside` the sphere, which will be
  good for ray-sphere computation(forbid nan).
- precache: If True, will precache all the rays for all pixels at once.
- device: If set to gpu, it will put data directly to gpu(which fasten the speed of ray generation, but takes gpu).
- pc_radius(base_3d_pc_dataset): Remove point cloud that are outside such absolute radius(all scaled by extra `1.05`).
- align_cam: Sometimes it can be used to align cam in a horizontal way.
- cam_t_offset: a list of 3 that adjust the c2w translation manually. This moves the object to center, good for neus geometric init.
- exchange_coord: Flexible to exchange/flip the coord to a standard system.
- eval_max_sample: To keep the closest-to-center_pose samples in the split.
Done after camera `scale_radius`. The radius is restricted within `scale_radius` range.
- ndc_space/center_pixel/normalize_rays_d: Affect the ray sampling function in `get_rays`.
## Augmentation:
The augmentation is for all image process in all time.
- n_rays: Sample `n_rays` instead of using all. But calling it every time may sample overlapping rays, not use in train.
- shuffle: shuffle all the rays from the same image together
- blend_bkg_color: Merge custom background to the image. Must have `mask` in inputs key.
- transfer_rgb: transfer the rgb space
## rgb and mask
- All color are in `rgb` order and normed by `255` into `0~1` range.
- All masks should be binary masks with `{0,1}` values. Can be [] if not exist.
## cameras
- cameras: a list of `render.camera.PerspectiveCamera`, used for ray generation.
- For every dataset, you should setup cameras by reading their own `c2w` and `intrinsic`
## rays:
- rays are generated by cameras, precache all rays if needed.
- `get_rays` generate rays in `(wh)` flatten order, but if you read by cv2 and reshape, img will be in `(hw)` order.
You need to get_rays by setting `wh_order=False` to change the order.
    - ndc_space: makes the rays in ndc space. You need to specify the near distance as well.
    - center_pixel: If set True, use (0.5, 1.5,...) instead of (0, 1, 2) as the pixel location.
    - normalize_rays_d: By default True to normalize the rays. but in some model we don't want it normalized.
## point cloud
- pts: `(n_pts, 3)` in world coordinate, `xyz` order
- color: `(n_pts, 3)`, `rgb` order, should be normed into `0~1` range.
- vis: `(n_cam, n_pts)`, visibility of each point in each cam. `{0,1}` values.
- pts is required, color/vis is optional.
## bounds
- a list of bounds in `(2, )` dim representing the near, far zvals. It will be expanded to match each ray.
  - You can directly set `(hw, 3)` bounds for each image.
- If exists, help the ray sampling in modeling progress. Can be [] if not exist.
- For the near/far, you can also set in `cfgs.rays.near/far`, or use ray-sphere intersection
for near/far calculation by setting `cfgs.rays.bounding_radius`.
- Since we use normalized rays for each image, it means that the given the same distance, the ray in center is extended
further than corner ones. It makes the sampling in roughly a sphere, when the cameras aligned on sphere surface.
If you want all the rays extended the same z-distance, you should not normalize the rays, this may be good for the case
of large scene but not object.

------------------------------------------------------------------------
# Dataset Class
Below are supported dataset class.

------------------------------------------------------------------------
## Capture
This class provides dataset from your capture data.
You need to run colmap to extract corresponding poses and train.
### Video_to_Image
If you capture video, you can follow `scripts/data_process.sh` to use `extract_video` to
extract video into images. Will write data to `cfgs.dir.data_dir/Capture/scene_name`
- video_path: actual video path
- scene_name: specify the scene name. Image will be written to `cfgs.dir.data_dir/Capture/scene_name/images`.
- video_downsample: downsample video frames by such factor. By default `1`.
- image_downsample: downsample each frame by such factor. By default `1`.
We suggest you to run extract/colmap on full image size and length to allow more accurate pose estimation. You can reduce
the image for training by setting config in dataset.
### Colmap Run_poses
You should install [colmap](https://colmap.github.io/) by yourself. We provide python script for processing.
you can follow `scripts/data_process.sh` to use `run_poses` to get colmap with poses and dense reconstruction.
Will write data to `cfgs.dir.data_dir/Capture/scene_name`
- match_type: `sequential_matcher` is good for sequential ordered images.
              `exhaustive_matcher` is good for random ordered image.
- dense_reconstruct: If true, run dense_reconstruct and get dense point cloud and mesh.
### Mask/Segmentation estimation
- TODO: We may add it in the future.
- But for now you can use external tools to extract the mask, and put it under `mask/xxx.png` in the data_dir.
### Volume regression from mask
If mask is provided, we use all masks in training set to get the 2d bbox, and simply regress a 3d volume for the object.
This help to get the close approximation of the bounding volume, which could be helpful in modelling for some explicit
methods.
- Run `python tools/get_3d_bbox_from_silhouette.py --configs configs/datasets/DATASET_NAME/SCENE_NAME.yaml`, you should
prepare a yaml file for this dataset and scene. It will output the regressed volume information and visualization of
volume and bbox.
- TODO: It is still a developing function. We plan to make the estimation more closely bounding to the object, and set
the pruning volume as model initialization.
### Dataset
Use `Capture` class for this dataset. It is specified by scene_name.
- scene_name: scene_name that is the folder name under `Capture`. Use it to be identifier.
#### Processing
Since we need to rescale the point_cloud and cam so that object(pc) is centered at (0,0,0). If we directly set pc.mean()
as (0,0,0), noise not on object will make the center incorrect. We do the following:
- Use all camera and ray from center image plane to get a closely approximate common view point,
which is close to object center, adjust cam/pc by this offset. This is optional by setting `center_by_view_dirs=True`.
- `center_by_view_dirs` do not gives out deterministic results all the time, which leads to inconsistency between train/eval. Only for checking.
- Norm cam and point by `scale_radius` to make them within a sphere with known range.
- Filter point cloud by `pc_radius` and remove point outside.
- Recenter cam and point by setting the filtered point cloud center as (0,0,0).
- Re-norm cam and point again to make cam on the surface of sphere with `scale_radius` and obj is centered.
- cam_t_offset: use to shift all the cam positions by -offset.
- test_holdout: is used for separating the train/test images.

- We test and show that the method is robust to make the coordinate system such that object is centered at (0,0,0),
cam is on surface with `scale_radius`. Only scale and translation is applied, do not affect the intrinsic.

File Structure is:
```
Capture
└───qqtiger
│     └───images
│     │    └─── *.png
│     └───images
│           └─── *.png
│     └───colmap_output.txt
│     └───database.db
│     └───poses_bounds.npy
│     └───sparse
│           └─── 0
│               └─── cameras.bin
│               └─── images.bin
│               └─── points3D.bin
│               └─── project.ini
└───etc(other scenes)
```

------------------------------------------------------------------------
## Standard benchmarks

### NeRF
Commonly used synthetic dataset. Specified by scene_name, read image/camera.
Since NeRF split the dataset into train/val/eval, we load all the camera from all split together, process cameras, and keep the cam for any split. This make the
transformation of camera(like norm_pose) consistent over all split.
- scene_name: scene_name that is the folder name under `NeRF`. Use it to be identifier.
- poses: for the poses, we do transform so that it matches the coord system in our proj.
  - Poses in all split are read and processed together for consistency.
- images: The image are in `RGBA` channels, needs to blend rgb by alpha. You can get mask from `alpha`.

```
NeRF
│   README.txt
└───lego
│     └───train
│     │    └─── r_*.png
│     └───val
│     │    └─── r_*.png
│     └───test
│     │    └─── r_*.png
│     │    └─── r_*_depth_*.png
│     │    └─── r_*_normal_*.png
│     └───transforms_train.json
│     └───transforms_val.json
│     └───transforms_test.json
└───chair
└───ship
└───etc(other scenes)
```

Ref: https://github.com/bmild/nerf

### LLFF
This is a forward facing dataset. Not object extraction is performed. Only used to view synthesis.
For fair comparison, test/val images have not overlapping with train images.
- The cameras are aligned flatten. Adjust the poses/bounds by range to avoid large xyz values.
- scene_name: scene_name that is the folder name under `LLFF`. Use it to be identifier.
- test_holdout: is used for separating the train/test images.
- NDC: We support NDC Space conversion if you set `ndc_space=True` in dataset.
  - In the original implementation, even they use `ndc_space` rays for sampling, the view_dirs sent to radianceNet is
still in non-ndc space. We don't follow it here but only used `ndc rays_d` as view_dirs. But notice that this affects
the performance, use `non-ndc rays_d` gets better result.

```
LLFF
└───fern
│     └───images
│     │    └─── *.JPG
│     └───images_4
│     │    └─── *.JPG
│     └───images_8
│     │    └─── *.JPG
│     └───mpis4
│     └───sparse
│          └─── 0
│               └─── cameras.bin
│               └─── images.bin
│               └─── points3D.bin
│               └─── project.ini
│     └───database.db
│     └───poses_bounds.npy
│     └───simplices.npy
│     └───trimesh.png
└───flower
└───horns
└───etc(other scenes)
```

Ref: https://github.com/Fyusion/LLFF & https://github.com/bmild/nerf

### DTU
Good for object reconstruction. Specified by scan_id, read image/mask/camera.
We download the version process by the author of NeuS(total 15 scans).
For fair comparison, test/val images have not overlapping with train images.(Just like `LLFF`)
- scan_id: int num for item selection. Use it to be identifier.
- test_holdout: is used for separating the train/test images.

```
DTU
└───dtu_scan65
│     └───image
│     │    └─── *.png
│     └───mask
│     │    └─── *.png
│     └───cameras_large.npz
│     └───cameras_sphere.npz  (We use this one)
└───dtu_scan24
└───dtu_scan37
└───etc(other scenes)
```

Ref: https://github.com/lioryariv/idr/blob/main/code/datasets/scene_dataset.py

### BlendedMVS
Good for object reconstruction. Specified by scene_name, read image/camera.
We download the version process by the author of VolSDF(total 9 scans).
For fair comparison, test/val images have not overlapping with train images.(Just like `LLFF`)
- scene_name: scene_name that is the folder name under `BlendedMVS`. Use it to be identifier.
- test_holdout: is used for separating the train/test images.
- In some case it uses `align_cam` and `exchange_coord` for changing the coordinate into a standard one.

```
BlendedMVS
└───scan1
│     └───image
│     │    └─── *.jgp
│     └───cameras.npz
└───scan2
└───scan3
└───etc(other scenes)
```

Ref: https://github.com/YoYo000/BlendedMVS

### NSVF
Similar synthetic dataset like NeRF. Specified by scene_name, read image/camera.
Since NSVF split the dataset into train/val/eval, we load all the camera from all split together, process cameras, and keep the cam for any split. This make the
transformation of camera(like norm_pose) consistent over all split.
- scene_name: scene_name that is the folder name under `NSVF`. Use it to be identifier.
- poses: for the poses, we do transform so that it matches the coord system in our proj.
  - - Poses in all split are read and processed together for consistency.
- images: The image are in `RGBA` channels, needs to blend rgb by alpha. You can get mask from `alpha`.

```
NSVF
│   README.txt
└───Wineholder
│     └───rgb
│     │    └─── 0_cam_train_*.png
│     │    └─── 1_cam_val_*.png
│     │    └─── 2_cam_test_*.png
│     └───pose
│     │    └─── 0_cam_train_*.txt
│     │    └─── 1_cam_val_*.txt
│     │    └─── 2_cam_test_*.txt
│     └───bbox.txt
│     └───intrinsics.txt
└───Bike
└───Robot
└───etc(other scenes)
```

Ref: https://lingjie0206.github.io/papers/NSVF/

### MipNerf-360
360 self-captured data. Like `LLFF`, processed with `Colmap`.
For fair comparison, test/val images have not overlapping with train images.(Just like `LLFF`)
- scene_name: scene_name that is the folder name under `MipNeRF360`. Use it to be identifier.
- test_holdout: is used for separating the train/test images.
- skip: different from `LLFF`, skip here is used to skip the final train/test images for fast image loading.

```
MipNeRF360
└───garden
│     └───images
│     │    └─── *.JPG
│     └───images_2
│     │    └─── *.JPG
│     └───images_4
│     │    └─── *.JPG
│     └───images_8
│     │    └─── *.JPG
│     └───sparse
│     │    └─── 0
│     │         └─── cameras.bin
│     │         └─── images.bin
│     │         └─── points3D.bin
│     └───poses_bounds.npy
└───counter
└───bicycle
└───etc(other scenes)
```

Ref: https://jonbarron.info/mipnerf360/

### Tanks and Temples
Captured outdoor 360 scene. A large object is at the center. Colmap processed poses and point_cloud is available
for 3d reconstruction. The official [link](https://www.tanksandtemples.org/) do not give correct intrinsic, so we use
the one from [nerf++](https://github.com/Kai-46/nerfplusplus). It splits train/val/test already and contains 4 scenes.
- scene_name: scene_name that is the folder name under `TanksAndTemples`. Use it to be identifier.
- ply: We do not load the `.ply` file for this moment.

```
TanksAndTemples
└───tat_training_Truck
│     └───train
│     │    └─── rgb
│     │    │     └─── *.png
│     │    └─── pose
│     │    │     └─── *.txt
│     │    └─── intrinsics
│     │    │     └─── *.txt
│     └───test
│     │    └─── rgb
│     │    │     └─── *.png
│     │    └─── pose
│     │    │     └─── *.txt
│     │    └─── intrinsics
│     │    │     └─── *.txt
│     └───pointcloud_norm.ply
│     └───camera_path
└───tat_intermediate_Train
│     └───train
│     └───validation  (Only Truck does not have validation, it just a subset of test)
│     └───test
└───tat_intermediate_Playground
└───tat_intermediate_M60
```

Ref: https://www.tanksandtemples.org/

### RTMV
High quality synthetic dataset with objects. More than 300 scenes are provided, but we only download the
`40_scenes` from 4 splits(`google_scanned`/`bricks->lego`/`amazon_berkely`/`abc`).
- scene_name: scene_name that is the folder name under `TanksAndTemples`. Use it to be identifier.
  - The scene_name should be in the form of `split_name/scene_name` like `google_scanned/00000`
- testhold: is used for separate the train/test images like `LLFF`.
- skip: different from `LLFF`, skip here is used to skip the final train/test images for fast image loading.
- ply: We do not load the `.ply` file for this moment.

```
TanksAndTemples
└───google_scanned
│     └───00000
│     │    └─── *.exr
│     │    └─── *.depth.exr
│     │    └─── *.seg.exr
│     │    └─── *.json
│     └───00001
│     └───(other scenes)
└───lego
│     └───Fire_temple
│     └───V8
│     └───(other scenes)
└───amazon_berkely
└───abc
```

Ref: http://www.cs.umd.edu/~mmeshry/projects/rtmv/


### Download address
Some datasets may not be downloaded from their ref address. We show all the address we use here.
- NeRF/LLFF: https://drive.google.com/drive/folders/128yBriW1IG_3NJ5Rp7APSTZsJqdJdfc1
- DTU: https://drive.google.com/drive/folders/1Nlzejs4mfPuJYORLbDEUDWlc9IZIbU0C
- BlenderMVS: https://www.dropbox.com/sh/oum8dyo19jqdkwu/AAAxpIifYjjotz_fIRBj1Fyla?preview=BlendedMVS.zip
- NSVF: https://github.com/facebookresearch/NSVF
- RTMV: https://drive.google.com/drive/folders/1cUXxUp6g25WwzHnm_491zNJJ4T7R_fum
- Tanks&Temples: https://www.tanksandtemples.org/download/; https://github.com/Kai-46/nerfplusplus
- MipNerf-360: http://storage.googleapis.com/gresearch/refraw360/360_v2.zip

------------------------------------------------------------------------
# Train/Val/Eval/Inference

## Train
Use all images for training. Same resolution as required.

For training, we should read all rays from all images together, shuffle each image pixels(rays), shuffle images,
concat all the image, sample in batch with `n_rays`. Each epoch just trained on batch. When all rays from all images
have been chosen, shuffle again.

For training, `scheduler` will handle special requirement in each shuffle of all rays. (See [trainer](./trainer.md) for detail.)

## Val
Use all images for validation, downscaled by 2/4 depends on shape.

Each valid epoch just input one image for rendering, so the batch_size for val is cast to be 1.

## Eval
Use several closest camera(to avg_cam) for metric evaluation,

use same resolution(Or scale if image really too large), and use custom cam paths for rendering video

- eval_batch_size: batch size for eval
- test_holdout: In order to split train/eval images, use this to holdout testset, by default 8.
only those will be fully rendered can calculate metric.

For some standard benchmark(`NeRF`/`NSVF`/`LLFF`), we follow the same split as official repos used for fair comparison.

------------------------------------------------------------------------
## Inference
Inference will be performed based on eval dataset params(intrinsic, img shape). If you do not set
the eval dataset, inference will not be performed.

###
to_gpu: If you set this to True, will create rays tensors on gpu directly if you use gpu.

### Render
Controls the params of render novel view(volume rendering), like the camera path. Check `geometry/poses.py` for detail.
  - type: list render cam move type. Support `circle`/`spiral`/`regular`/`swing`.
  - n_cam: list of cam for each type
  - repeat: list of repeat num for each type. In case you don't want to render repeatedly(in `swing`/`circle`).
  - radius: radius of cam path, single value
  - u_start/u_range/v_ratio/v_range/normal/n_rot/reverse: for placing the cameras. Check `poses` for details.
  - fps: render video fps.
  - center_pixel: Whether to generate the rays from `center_pixel` rather than corner.

#### Surface_render
If you set this, also render the view by finding the surface pts and render. Good for sdf models like Neus and volsdf.
- chunk_rays_factor: In surface_render mode, you can progress more rays in a batch, set a factor to allow large rays batch.
- method: method to find the surface pts. Support `sphere_tracing`(sdf)/and `secant_root_finding`(any).
- n_step/n_iter/threshold: params to find root. Check the `geometry/rays.py` for detail.
- level/grad_dir: Determine the value flow. SDF is 0.0/ascent(inside smaller), density is +level/descent(inside larger).

### Volume
Controls the params of volume estimation and mesh extraction/rendering.

We support extract the mesh from volume field and getting the colors of mesh/verts by normal direction.

Remaining that this function is not always actually in `NeRF` like rendering methods, since they are not developed
for mesh extraction. We plan to merge other mesh extraction methods.

- origin/n_grid/side/xyz_len: params for volume position and size. Check `geometry/volume.py` for detail.
- level/grad_dir: Determine the value flow. SDF is 0.0/ascent(inside smaller), density is +level/descent(inside larger).
- chunk_pts_factor: In extract_mesh mode, you can progress more pts in a batch, set a factor to allow large pts batch.
- render_mesh: For rasterization of mesh only. You can use `pytorch3d` or `open3d` as backend.
