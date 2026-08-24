# 空壳工作流扫描（empty-shells-v2）

- 扫描目标: `corpus/decoded`
- 扫描数: **581**（platform id 去重后唯一工作流）
- 生成: 2026-08-24（`tools/empty_shell_scan.py`）

## 判定标准

业务节点 = 节点谱排除结构噪音 `{start,end,comment,output}`（全量递归,
容器 batch/loop 嵌套子图计入: yaml 容器键 `nodes`, legacy json 容器键
`nodes/children/blocks`）。

| verdict | 判据 | 数量 |
| --- | --- | --- |
| HARD | 业务节点==0 且无 output（纯 start/end/comment 骨架） | 3 |
| OUTPUT-ONLY | 业务节点==0 但有 output（仅输出骨架） | 0 |
| MINIMAL | 业务节点==1（单执行节点，近乎空壳） | 14 |
| NORMAL | 其余（有实质业务） | 564 |
| DECODE-FAIL | 解码失败（无法判定） | 0 |

> 注: 早期报告「262 个空壳」是误读 —— 262 是 standard 信封格式计数
> （corpus-v1），且 `sub: []` 只表示无子工作流调用，不是空壳判据。
> 本清单是权威口径。

## 空壳 / 弱壳清单

| id | name | format | total | biz | verdict | profile |
| --- | --- | --- | --- | --- | --- | --- |
| 7531671963719876647 | X168_upload_img_370_1 | json | 2 | 0 | HARD | end:1 start:1 |
| 7531683265519517735 | X242_flux_video_1 | json | 2 | 0 | HARD | end:1 start:1 |
| 7542708432375382016 | File2URL | yaml | 2 | 0 | HARD | end:1 start:1 |
| 7511583026155307060 | ZW10 | json | 3 | 1 | MINIMAL | end:1 plugin:1 start:1 |
| 7531363041076019219 | X58_suno2_gen_mp3_1 | json | 5 | 1 | MINIMAL | comment:2 end:1 plugin:1 start:1 |
| 7531366040989024265 | X77_Ptupianzhuangif_picture_1 | json | 4 | 1 | MINIMAL | end:1 output:1 plugin:1 start:1 |
| 7531637465481723950 | X94_Pshuji_picture_1 | json | 3 | 1 | MINIMAL | end:1 llm:1 start:1 |
| 7531645755266777140 | X148_Vvoice_clone_489_1 | json | 3 | 1 | MINIMAL | end:1 plugin:1 start:1 |
| 7531650135755964426 | X153_Vvidu_video_task_info02_1 | json | 5 | 1 | MINIMAL | comment:2 end:1 plugin:1 start:1 |
| 7531671860528267299 | X167_Ttaobao_data_421_1 | json | 4 | 1 | MINIMAL | comment:1 end:1 plugin:1 start:1 |
| 7531672716300402740 | X175_Pupdate_img_all_1 | json | 3 | 1 | MINIMAL | end:1 plugin:1 start:1 |
| 7531672907379359796 | X177_P_updata_img_picture_1 | json | 3 | 1 | MINIMAL | end:1 plugin:1 start:1 |
| 7531673002464575540 | X178_S_search_2_buy_407_1 | json | 4 | 1 | MINIMAL | comment:1 end:1 plugin:1 start:1 |
| 7531673741219528750 | X184_removebg_499_1 | json | 4 | 1 | MINIMAL | end:1 output:1 plugin:1 start:1 |
| 7531674503286947903 | X189_Wys_text_1 | json | 3 | 1 | MINIMAL | end:1 llm:1 start:1 |
| 7531676246083731456 | X199_W_xhs_get_user_note_1 | json | 5 | 1 | MINIMAL | comment:2 end:1 plugin:1 start:1 |
| 7531677105454432308 | X204_V_down_video_1 | json | 5 | 1 | MINIMAL | comment:2 end:1 plugin:1 start:1 |
