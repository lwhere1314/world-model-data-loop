# Schemas

这里定义最小可用字段，方便自动化生产、curation、evaluation 接起来。

## 1. Protocol Manifest

```csv
protocol_id,task_family,scene_seed,target_object,stressors,duration_s,num_steps,prompt_path,action_schedule_path,success_contract
```

字段说明：

| 字段 | 含义 |
| --- | --- |
| `protocol_id` | 唯一任务 ID |
| `task_family` | 任务族，例如 memory_consistency、object_interaction、navigation |
| `scene_seed` | 初始图、视频、3D scene 或文字场景 |
| `target_object` | 要追踪或交互的目标 |
| `stressors` | 难点列表 |
| `duration_s` | rollout 时长 |
| `num_steps` | action step 数 |
| `prompt_path` | shared prompt |
| `action_schedule_path` | action list |
| `success_contract` | 成功标准 |

## 2. Rollout Manifest

```csv
rollout_id,protocol_id,model_name,model_version,runner_version,seed,video_path,metadata_path,status,failure_reason
```

`status` 可选：

- `generated`
- `invalid_video`
- `control_failed`
- `ready_for_eval`
- `hard_negative`
- `needs_human_review`

## 3. Action Schedule

```csv
step,start_s,end_s,action,operator_instruction,expected_target_visible,expected_state_delta,purpose
```

示例 action：

- `HOLD`
- `YAW_R`
- `YAW_L`
- `FWD`
- `BACK`
- `LEFT`
- `RIGHT`
- `INTERACT`
- `LOOK_AT`
- `PICK`
- `PLACE`

`purpose` 很重要。它解释这一步为什么存在，例如：

```text
build initial memory
turn away from target
create long offscreen gap
pass a similar confuser
return for final verification
```

## 4. Curation Record

```csv
rollout_id,valid_video,initial_state_valid,action_following_valid,duplicate_group,coverage_tags,curation_status,hard_negative_type,review_priority,notes
```

## 5. Frame Label

```csv
rollout_id,frame_ts,frame_path,target_visible,target_identity,text_consistency,layout_consistency,geometry_drift,scale_drift,confuser_present,notes
```

评分建议使用 `0/1/2/3`：

- `0`: fail
- `1`: weak
- `2`: mostly correct
- `3`: correct and stable

## 6. Segment Label

```csv
rollout_id,segment_id,start_s,end_s,phase,revisit_valid,action_following,text_consistency,storefront_or_object_consistency,spatial_consistency,failure_type,notes
```

`phase` 示例：

- `initial_observe`
- `turn_away`
- `move_away`
- `confuser`
- `return`
- `final_revisit`
- `interaction`

## 7. Trajectory Label

```csv
rollout_id,protocol_id,model_name,valid_video,action_following_score,memory_consistency_score,physical_plausibility_score,task_success,pass_fail,uncertainty,human_summary
```

## 8. Evaluation Request JSON

```json
{
  "request_id": "",
  "protocol_id": "",
  "rollout_id": "",
  "model_name": "",
  "task_family": "",
  "prompt": "",
  "action_schedule": [],
  "video_path": "",
  "keyframes": [],
  "contact_sheet": "",
  "fewshot_anchors": [
    {
      "example_id": "",
      "label": "",
      "reason": ""
    }
  ],
  "required_scores": [
    "valid_video",
    "action_following_score",
    "memory_consistency_score",
    "physical_plausibility_score",
    "task_success",
    "uncertainty"
  ]
}
```

## 9. Evaluation Output JSON

```json
{
  "request_id": "",
  "valid_video": true,
  "action_following_score": 0,
  "target_identity_score": 0,
  "text_consistency_score": 0,
  "layout_consistency_score": 0,
  "physical_plausibility_score": 0,
  "memory_consistency_score": 0,
  "task_success": false,
  "pass_fail": "fail",
  "failure_type": [],
  "uncertainty": 0,
  "evidence": "",
  "frames_to_review": []
}
```

## 10. Few-shot Anchor

```json
{
  "example_id": "",
  "task_family": "",
  "media_path": "",
  "label_level": "frame|segment|trajectory|exploration",
  "human_label": "",
  "score": 0,
  "failure_type": [],
  "why_human_judges_this_way": "",
  "what_agent_should_learn": ""
}
```

这个字段最关键：

```text
what_agent_should_learn
```

它不是普通标签，而是把人类判断转成 agent 的探索和评价偏好。
