# Adjustments to the original codebase and usage with custom data
## 4 Features Waymo

The following changes were made to use the waymo dataset with the four classical features x, y, z, and intensity and neglect the fifth feature elongation, which is specific to the LiDARs used by waymo:

NumPointFeatures changed to 4 \
https://github.com/dlengerer/CenterPointCustomData/blob/7876b5926602fea5b6aa526d8a2ba379ddd237e6/det3d/datasets/waymo/waymo.py#L20

added [:, :4] to cut the fifth feature when loading a waymo frame \
https://github.com/dlengerer/CenterPointCustomData/blob/7876b5926602fea5b6aa526d8a2ba379ddd237e6/det3d/datasets/pipelines/loading.py#L78

added a variable "randomlySample" to toggle the sampling of pedestrians and vehicles: \
https://github.com/dlengerer/CenterPointCustomData/blob/7876b5926602fea5b6aa526d8a2ba379ddd237e6/det3d/datasets/utils/create_gt_database.py#L89-L104 \
The variable is set to the original behavior of sampling the objects. It can be deactivated by using: \
```
python tools/create_data.py waymo_data_prep --root_path=data/Waymo --split train --nsweeps=1 --randomlySample=False
```

It is only necessary when using custom data.

## Custom Data
To use custom data to train, fine-tune, or test with CenterPoint, the custom data was converted in the same format as the waymo dataset. It may not be the cleanest solution and there sure are some other possible ways to go, but for me it was the easiest and fastest solution with the slightest code adjustments. The code snippets are mostly for understanding and have to be adapted for your own data.

For the custom data to be used, the lidar data has to be in the classical lidar coordinate system, x facing forward, y facing left, and z upwards. Intensity needs to be normalized to (0 - 1). For each frame, a pickle file needs to be generated:

```python
def save_pcl(x, y, z, intensity, seq_id, frame_id):
    frame_dict = {
        'scene_name' : 'somescenename_' + str(seq_id),
        'frame_name' : 'someframename_' + str(frame_id).zfill(5),
        'frame_id' : frame_id,
        'lidars' : {
            'points_xyz' : np.column_stack((x, y, z)),
            'points_feature' : np.column_stack((intensity, np.zeros((len(x), 1), dtype=np.float32)))
        }
    }
  
    path_to_file = os.path.join('somepath/dataset/train/lidar', 'seq_' + str(seq_id) + '_frame_' + str(frame_id) + '.pkl')

    with open(path_to_file, 'wb') as f:
          pickle.dump(frame_dict, f)
```

The annotations bounding boxes need to be in the waymo coordinates as described here: \
https://github.com/dlengerer/CenterPoint/blob/7876b5926602fea5b6aa526d8a2ba379ddd237e6/det3d/datasets/waymo/waymo_common.py#L266-L268

For each object, an object dictionary has to be created, and all these object dictionaries are collected in a list:

```python
objects = []

# for loop for understanding, the iteration of the objects has to be implemented data specific
for x in range(num_objects):
    object_dic = {
        'id' : class_key + '_' + str(idx),
        'name' : class_key + '_' + str(idx),
        'label' : CAT_NAME_TO_ID[class_key],                    # car == 1, pedestrian == 2, cyclist == 4
        'num_points' : num_points,                              # has to be correct if detection_level is important, else > 0
        'detection_difficulty_level' : 0,                       # will be caclulated based on num_points if zero
        'global_speed' : np.zeros((1,2), dtype=np.float32),     # not used for detection (as far as i can tell)
        'global_accel' : np.zeros((1,2), dtype=np.float32),     # not used for detection (as far as i can tell)
        'box' : [xctr, yctr, zctr, xlen, ylen, zlen, zrot]
    }

    objects.append(object_dic)
```

Finally a pickle file has to be generated for the annotations of each frame:

``` python
def save_annots(objects, seq_id, frame_id):
    annots_dic = {
        'scene_name' : 'somescenename_' + str(seq_id),
        'frame_name' : 'someframename_' + str(frame_id).zfill(5),
        'frame_id' : frame_id,
        'veh_to_global' : np.identity(4),                       # not used for detection (as far as i can tell)
        'objects' : objects,
    }

    path_to_file = os.path.join('somepath/dataset/train/annos', 'seq_' + str(seq_id) + '_frame_' + str(frame_id) + '.pkl')

    with open(pth, 'wb') as f:
        pickle.dump(annots_dic, f)
```

From there it is the same as with a waymo dataset, use the same commands starting from section "Create info files" (with adapted root_path) [WAYMO](WAYMO.md). Afterwards your folder structure should be similar to the one shown in the Waymo Readme.

## Contact
Any questions or suggestions are welcome! \
Daniel.Lengerer@hs-augsburg.de