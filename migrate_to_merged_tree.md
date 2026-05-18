下面给你一份 **UOS / Linux 迁移脚本**，用于把两个源目录中的文件按最新目录体系迁移到已经建好的新目录中。

脚本特性：

* 支持两个源目录：`SRC1`、`SRC2`
* 支持目标目录：`DST`
* 支持 `--dry-run` 预演，不实际移动
* 遇到重名文件不覆盖
* 同名同哈希文件移动到 `99_重复_临时_待清理/01_疑似重复_待哈希确认/同名同哈希重复`
* 同名但内容不同的文件自动追加来源后缀
* 未命中规则的文件统一进入 `99_重复_临时_待清理/07_来源不明文件/未命中_SRC1` 或 `未命中_SRC2`
* 生成迁移日志

---

# 一、UOS / Linux 迁移脚本

保存为：

```bash
migrate_to_merged_tree.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -lt 3 ]; then
    echo "用法："
    echo "  $0 <源目录1> <源目录2> <目标目录> [--dry-run]"
    echo
    echo "示例："
    echo "  $0 \"/home/user/文件夹1\" \"/home/user/文件夹2\" \"/home/user/长江电力生产经营管理数字化平台资料库_合并\" --dry-run"
    echo "  $0 \"/home/user/文件夹1\" \"/home/user/文件夹2\" \"/home/user/长江电力生产经营管理数字化平台资料库_合并\""
    exit 1
fi

SRC1="$1"
SRC2="$2"
DST="$3"
DRY_RUN="${4:-}"

if [ ! -d "$SRC1" ]; then
    echo "源目录1不存在：$SRC1"
    exit 1
fi

if [ ! -d "$SRC2" ]; then
    echo "源目录2不存在：$SRC2"
    exit 1
fi

if [ ! -d "$DST" ]; then
    echo "目标目录不存在，请先创建目录体系：$DST"
    exit 1
fi

TS="$(date +%Y%m%d_%H%M%S)"
LOG_DIR="$DST/99_重复_临时_待清理/07_来源不明文件"
LOG_FILE="$LOG_DIR/迁移日志_$TS.log"

mkdir -p "$LOG_DIR"

log() {
    local msg="$1"
    echo "$msg"
    echo "$msg" >> "$LOG_FILE"
}

run_cmd() {
    if [ "$DRY_RUN" = "--dry-run" ]; then
        log "[DryRun] $*"
    else
        "$@"
    fi
}

ensure_dir() {
    local dir="$1"
    if [ "$DRY_RUN" = "--dry-run" ]; then
        log "[DryRun] mkdir -p \"$dir\""
    else
        mkdir -p "$dir"
    fi
}

hash_file() {
    sha256sum "$1" | awk '{print $1}'
}

unique_path() {
    local dir="$1"
    local filename="$2"
    local label="$3"

    local base ext name candidate counter

    base="$filename"

    if [[ "$base" == *.* ]]; then
        name="${base%.*}"
        ext=".${base##*.}"
    else
        name="$base"
        ext=""
    fi

    candidate="$dir/$base"
    counter=1

    while [ -e "$candidate" ]; do
        candidate="$dir/${name}__${label}_${counter}${ext}"
        counter=$((counter + 1))
    done

    echo "$candidate"
}

safe_move_file() {
    local src_file="$1"
    local dst_dir="$2"
    local label="$3"

    [ -f "$src_file" ] || return 0

    ensure_dir "$dst_dir"

    local filename dst_file src_hash dst_hash duplicate_dir duplicate_target final_target

    filename="$(basename "$src_file")"
    dst_file="$dst_dir/$filename"

    if [ ! -e "$dst_file" ]; then
        log "[MOVE] \"$src_file\" -> \"$dst_file\""
        if [ "$DRY_RUN" != "--dry-run" ]; then
            mv "$src_file" "$dst_file"
        fi
        return 0
    fi

    src_hash="$(hash_file "$src_file")"
    dst_hash="$(hash_file "$dst_file")"

    if [ "$src_hash" = "$dst_hash" ]; then
        duplicate_dir="$DST/99_重复_临时_待清理/01_疑似重复_待哈希确认/同名同哈希重复/$label"
        ensure_dir "$duplicate_dir"
        duplicate_target="$(unique_path "$duplicate_dir" "$filename" "$label")"

        log "[DUP-SAME-HASH] \"$src_file\" -> \"$duplicate_target\""
        if [ "$DRY_RUN" != "--dry-run" ]; then
            mv "$src_file" "$duplicate_target"
        fi
    else
        final_target="$(unique_path "$dst_dir" "$filename" "$label")"

        log "[DUP-DIFF-HASH] \"$src_file\" -> \"$final_target\""
        if [ "$DRY_RUN" != "--dry-run" ]; then
            mv "$src_file" "$final_target"
        fi
    fi
}

move_tree_contents() {
    local src_dir="$1"
    local dst_dir="$2"
    local label="$3"

    if [ ! -d "$src_dir" ]; then
        log "[SKIP] 源目录不存在：$src_dir"
        return 0
    fi

    log "[DIR] \"$src_dir\" -> \"$dst_dir\""

    while IFS= read -r -d '' file; do
        local rel subdir target_dir
        rel="${file#$src_dir/}"
        subdir="$(dirname "$rel")"

        if [ "$subdir" = "." ]; then
            target_dir="$dst_dir"
        else
            target_dir="$dst_dir/$subdir"
        fi

        safe_move_file "$file" "$target_dir" "$label"
    done < <(find "$src_dir" -type f -print0)

    if [ "$DRY_RUN" != "--dry-run" ]; then
        find "$src_dir" -depth -type d -empty -delete 2>/dev/null || true
    fi
}

move_root_files_by_rule_SRC1() {
    local root="$1"

    while IFS= read -r -d '' file; do
        local name dest

        name="$(basename "$file")"
        dest=""

        case "$name" in
            BIZBOK*|bizbok*|业务架构解构与实践*|企业架构及典型设计*|企业架构的数字化转型*|企业级业务架构设计*|华为企业架构设计方法及实例*)
                dest="$DST/09_外部参考与方法论资料/01_企业架构与业务架构方法论"
                ;;
            *数字化转型*|*IT架构蓝图*|德勤集团IT蓝图*|集团企业IT蓝图*|集团企业数字化转型*|华为数字化转型之道*|xx企业数字化转型项目顶层规划方案*)
                dest="$DST/09_外部参考与方法论资料/02_数字化转型与IT蓝图参考"
                ;;
            *Maximo*|maximo*|modulesandapplications*|火电厂及水电站生产管理*)
                dest="$DST/09_外部参考与方法论资料/03_EAM_Maximo_生产管理参考"
                ;;
            *SAP*|*ERP*|大型制造业企业数据架构顶层设计总体规划方案*)
                dest="$DST/09_外部参考与方法论资料/04_SAP_ERP_SGERP参考"
                ;;
            英诺森交流ppt.pdf|数字化转型及实践-集团专场*)
                dest="$DST/09_外部参考与方法论资料/05_咨询公司与行业案例"
                ;;
            199个经典逻辑思维工具框架模型*)
                dest="$DST/09_外部参考与方法论资料/06_工具框架与通用课件"
                ;;
            长江电力-brief.pdf)
                dest="$DST/08_制度标准与管理体系文件/08_公司背景与对外资料/01_长江电力公司介绍"
                ;;
            长电2024ESG.pdf)
                dest="$DST/08_制度标准与管理体系文件/08_公司背景与对外资料/02_ESG与社会责任"
                ;;
            *)
                dest="$DST/99_重复_临时_待清理/07_来源不明文件/源目录1_根目录散文件"
                ;;
        esac

        safe_move_file "$file" "$dest" "SRC1_ROOT"
    done < <(find "$root" -maxdepth 1 -type f -print0)
}

move_root_files_by_rule_SRC2() {
    local root="$1"

    while IFS= read -r -d '' file; do
        local name dest

        name="$(basename "$file")"
        dest=""

        case "$name" in
            *UI规范*|*交互界面设计规范*)
                dest="$DST/06_数字底座与业务应用开发平台/05_UI与交互规范"
                ;;
            *)
                dest="$DST/99_重复_临时_待清理/07_来源不明文件/源目录2_根目录散文件"
                ;;
        esac

        safe_move_file "$file" "$dest" "SRC2_ROOT"
    done < <(find "$root" -maxdepth 1 -type f -print0)
}

move_remaining_files() {
    local root="$1"
    local target="$2"
    local label="$3"

    while IFS= read -r -d '' file; do
        local rel subdir dst_dir

        rel="${file#$root/}"
        subdir="$(dirname "$rel")"

        if [ "$subdir" = "." ]; then
            dst_dir="$target"
        else
            dst_dir="$target/$subdir"
        fi

        safe_move_file "$file" "$dst_dir" "$label"
    done < <(find "$root" -type f -print0)

    if [ "$DRY_RUN" != "--dry-run" ]; then
        find "$root" -depth -type d -empty -delete 2>/dev/null || true
    fi
}

log "========== 资料库迁移开始 =========="
log "源目录1：$SRC1"
log "源目录2：$SRC2"
log "目标目录：$DST"
log "执行模式：${DRY_RUN:-正式移动}"
log "日志文件：$LOG_FILE"
log "==================================="

# ============================================================
# 源目录1：目录级迁移规则
# ============================================================

# 01_项目管理_规划_合同_招投标
move_tree_contents "$SRC1/0、早期规划" \
    "$DST/01_项目管理_规划_合同_招投标/01_早期规划与立项可研" \
    "SRC1_早期规划"

move_tree_contents "$SRC1/1、招投标文件" \
    "$DST/01_项目管理_规划_合同_招投标/03_招投标与采购文件" \
    "SRC1_招投标"

move_tree_contents "$SRC1/0、实施/合同边界" \
    "$DST/01_项目管理_规划_合同_招投标/04_合同边界与合同流程" \
    "SRC1_合同边界"

move_tree_contents "$SRC1/0、实施/进度计划" \
    "$DST/01_项目管理_规划_合同_招投标/05_进度计划与工作安排" \
    "SRC1_进度计划"

move_tree_contents "$SRC1/0、实施/会议纪要" \
    "$DST/01_项目管理_规划_合同_招投标/06_阶段汇报与评审材料/会议纪要" \
    "SRC1_会议纪要"

move_tree_contents "$SRC1/0、实施/蓝图设计/2辉总" \
    "$DST/01_项目管理_规划_合同_招投标/06_阶段汇报与评审材料/2辉总" \
    "SRC1_阶段汇报_2辉总"

# 02_项目实施与交付成果
move_tree_contents "$SRC1/0、实施/01调研阶段" \
    "$DST/02_项目实施与交付成果/01_实施管理与过程资料/01调研阶段" \
    "SRC1_实施_调研阶段"

move_tree_contents "$SRC1/0、实施/2-业务设计交付物模板（已校核完善）" \
    "$DST/02_项目实施与交付成果/02_交付物模板库/01_已解压模板" \
    "SRC1_交付物模板"

# 压缩包单独迁移
if [ -f "$SRC1/0、实施/2-业务设计交付物模板（已校核完善）.rar" ]; then
    safe_move_file "$SRC1/0、实施/2-业务设计交付物模板（已校核完善）.rar" \
        "$DST/02_项目实施与交付成果/02_交付物模板库/02_原始压缩包备份" \
        "SRC1_模板压缩包"
fi

# 交付文档整体进入交付成果区；后续可在目标目录内再细分当前推荐版/历史版本
move_tree_contents "$SRC1/0、实施/交付文档" \
    "$DST/02_项目实施与交付成果/06_交付成果数据表与清单/交付文档原始归集" \
    "SRC1_交付文档"

# 0、实施根目录下剩余过程文件
move_tree_contents "$SRC1/0、实施" \
    "$DST/02_项目实施与交付成果/01_实施管理与过程资料/0实施剩余过程资料" \
    "SRC1_0实施剩余"

# 03_业务架构与蓝图设计工作区
move_tree_contents "$SRC1/0、实施/蓝图设计" \
    "$DST/03_业务架构与蓝图设计工作区/08_蓝图设计过程材料" \
    "SRC1_蓝图设计"

# 04_调研与收资资料
move_tree_contents "$SRC1/8、现场调研材料-三峡电站/附件1 业务流程图" \
    "$DST/04_调研与收资资料/02_三峡电厂业务流程图_L2-L5" \
    "SRC1_业务流程图"

move_tree_contents "$SRC1/8、现场调研材料-三峡电站/音频" \
    "$DST/04_调研与收资资料/03_三峡电厂录音_录像_照片/20250623-0626_访谈录音" \
    "SRC1_调研音频"

move_tree_contents "$SRC1/8、现场调研材料-三峡电站/三峡电厂调研-生产管理-20260305" \
    "$DST/04_调研与收资资料/03_三峡电厂录音_录像_照片/20260305_生产管理照片" \
    "SRC1_生产管理照片"

move_tree_contents "$SRC1/8、现场调研材料-三峡电站/记录" \
    "$DST/04_调研与收资资料/01_三峡电厂现场调研/记录" \
    "SRC1_三峡调研记录"

move_tree_contents "$SRC1/8、现场调研材料-三峡电站/三峡电厂资料/技术标准" \
    "$DST/08_制度标准与管理体系文件/04_三峡电厂制度与标准/技术标准" \
    "SRC1_三峡技术标准"

move_tree_contents "$SRC1/8、现场调研材料-三峡电站/三峡电厂资料/现行制度及清单" \
    "$DST/08_制度标准与管理体系文件/04_三峡电厂制度与标准/现行制度及清单" \
    "SRC1_三峡制度"

move_tree_contents "$SRC1/8、现场调研材料-三峡电站/三峡电厂资料" \
    "$DST/04_调研与收资资料/01_三峡电厂现场调研/三峡电厂资料_剩余" \
    "SRC1_三峡资料剩余"

move_tree_contents "$SRC1/8、现场调研材料-三峡电站" \
    "$DST/04_调研与收资资料/01_三峡电厂现场调研" \
    "SRC1_三峡现场调研"

move_tree_contents "$SRC1/2、IFS原始资料（手册、截图、培训PPT）" \
    "$DST/04_调研与收资资料/04_IFS原始资料" \
    "SRC1_IFS"

# 05_ePMS及关联系统资料
move_tree_contents "$SRC1/10、下阶段/19个业务系统整合分析" \
    "$DST/05_ePMS及关联系统资料/02_19个业务系统整合分析" \
    "SRC1_19系统整合"

move_tree_contents "$SRC1/6、其他关联业务系统现状-历史资料/2-项目关联系统资料" \
    "$DST/05_ePMS及关联系统资料/07_其他关联系统历史资料" \
    "SRC1_关联系统历史"

move_tree_contents "$SRC1/6、其他关联业务系统现状-历史资料/管理信息系统现状-信通院" \
    "$DST/05_ePMS及关联系统资料/08_管理信息系统现状_信通院" \
    "SRC1_信通院"

move_tree_contents "$SRC1/6、其他关联业务系统现状-历史资料" \
    "$DST/05_ePMS及关联系统资料/07_其他关联系统历史资料/其他剩余" \
    "SRC1_关联系统剩余"

# 06_数字底座与业务应用开发平台
move_tree_contents "$SRC1/3、数字底座+应用平台（阿里相关材料）/20241219技术基础底座交付物" \
    "$DST/06_数字底座与业务应用开发平台/01_数字化技术基础底座交付物/20241219技术基础底座交付物" \
    "SRC1_数字底座20241219"

move_tree_contents "$SRC1/3、数字底座+应用平台（阿里相关材料）/长江电力EAM业数字化业务应用开发平台验收材料-阿里材料" \
    "$DST/06_数字底座与业务应用开发平台/02_业务应用开发平台验收材料/阿里材料" \
    "SRC1_应用开发平台验收"

move_tree_contents "$SRC1/3、数字底座+应用平台（阿里相关材料）/参考文档" \
    "$DST/06_数字底座与业务应用开发平台/06_参考文档_云巧_DDD_平台材料" \
    "SRC1_数字底座参考"

move_tree_contents "$SRC1/3、数字底座+应用平台（阿里相关材料）" \
    "$DST/06_数字底座与业务应用开发平台/06_参考文档_云巧_DDD_平台材料/其他剩余" \
    "SRC1_数字底座剩余"

# 07_后续项目建设
move_tree_contents "$SRC1/10、下阶段/数据质量" \
    "$DST/07_后续项目建设/05_数据质量" \
    "SRC1_数据质量"

move_tree_contents "$SRC1/10、下阶段/运行调度管理" \
    "$DST/07_后续项目建设/03_运行调度_值班_检修计划原型材料/运行调度管理" \
    "SRC1_运行调度"

move_tree_contents "$SRC1/10、下阶段/新建文件夹" \
    "$DST/07_后续项目建设/03_运行调度_值班_检修计划原型材料/调度管理原型" \
    "SRC1_调度管理原型"

move_tree_contents "$SRC1/10、下阶段/精益生产运行管理" \
    "$DST/07_后续项目建设/04_精益生产运行管理" \
    "SRC1_精益生产"

move_tree_contents "$SRC1/10、下阶段" \
    "$DST/07_后续项目建设/01_高级应用需求与建设路线" \
    "SRC1_后续项目"

# 08_制度标准与管理体系文件
move_tree_contents "$SRC1/4、外部参考资料" \
    "$DST/09_外部参考与方法论资料/03_EAM_Maximo_生产管理参考/外部参考资料原始归集" \
    "SRC1_外部参考"

# 09_外部参考与方法论资料
move_tree_contents "$SRC1/9、SGERP实施" \
    "$DST/09_外部参考与方法论资料/04_SAP_ERP_SGERP参考" \
    "SRC1_SGERP"

# 临时目录
move_tree_contents "$SRC1/0、实施/temp" \
    "$DST/99_重复_临时_待清理/06_临时截图与草稿/temp" \
    "SRC1_temp"

# 源目录1根目录散文件
move_root_files_by_rule_SRC1 "$SRC1"


# ============================================================
# 源目录2：目录级迁移规则
# ============================================================

# 01_项目管理_规划_合同_招投标
move_tree_contents "$SRC2/0、TGEAM项目前期咨询设计成果" \
    "$DST/01_项目管理_规划_合同_招投标/02_前期咨询设计成果_TGEAM" \
    "SRC2_TGEAM前期"

move_tree_contents "$SRC2/2-其他系统/数字化管理中心合同流程" \
    "$DST/01_项目管理_规划_合同_招投标/04_合同边界与合同流程/数字化管理中心合同流程" \
    "SRC2_合同流程"

# 04_调研与收资资料
move_tree_contents "$SRC2/11-三峡电厂收资（6月下旬调研）/两票管理操作" \
    "$DST/04_调研与收资资料/03_三峡电厂录音_录像_照片/两票管理操作" \
    "SRC2_两票管理操作"

move_tree_contents "$SRC2/11-三峡电厂收资（6月下旬调研）/安全智能管控系统-安监部（录屏照片）" \
    "$DST/04_调研与收资资料/03_三峡电厂录音_录像_照片/安全智能管控系统_安监部" \
    "SRC2_安全智能管控"

move_tree_contents "$SRC2/11-三峡电厂收资（6月下旬调研）/实验室智能检测及管控系统（计量 录屏）" \
    "$DST/04_调研与收资资料/03_三峡电厂录音_录像_照片/实验室智能检测及管控系统_计量" \
    "SRC2_计量录屏"

move_tree_contents "$SRC2/11-三峡电厂收资（6月下旬调研）/数据填报系统（可靠性 录屏）" \
    "$DST/04_调研与收资资料/03_三峡电厂录音_录像_照片/数据填报系统_可靠性" \
    "SRC2_可靠性录屏"

move_tree_contents "$SRC2/11-三峡电厂收资（6月下旬调研）/运行方式系统" \
    "$DST/04_调研与收资资料/03_三峡电厂录音_录像_照片/运行方式系统" \
    "SRC2_运行方式录屏"

move_tree_contents "$SRC2/11-三峡电厂收资（6月下旬调研）" \
    "$DST/04_调研与收资资料/05_其他现场调研材料/三峡电厂收资剩余" \
    "SRC2_三峡收资剩余"

# 05_ePMS及关联系统资料
move_tree_contents "$SRC2/1-ePMS系统/ePMS2024版_操作手册" \
    "$DST/05_ePMS及关联系统资料/01_ePMS系统资料/01_ePMS系统操作手册_2024版" \
    "SRC2_ePMS2024"

move_tree_contents "$SRC2/1-ePMS系统/公司EPMS相关材料-CD" \
    "$DST/05_ePMS及关联系统资料/01_ePMS系统资料/03_ePMS模块介绍材料" \
    "SRC2_ePMS模块介绍"

move_tree_contents "$SRC2/1-ePMS系统" \
    "$DST/05_ePMS及关联系统资料/01_ePMS系统资料/04_ePMS功能清单与系统关系/ePMS剩余" \
    "SRC2_ePMS剩余"

move_tree_contents "$SRC2/2017版操作手册和编码规范" \
    "$DST/05_ePMS及关联系统资料/01_ePMS系统资料/02_ePMS系统操作手册_2017版与编码规范" \
    "SRC2_ePMS2017"

move_tree_contents "$SRC2/2-其他系统/计划与预算管理示范应用" \
    "$DST/05_ePMS及关联系统资料/01_ePMS系统资料/05_生产经营管理示范应用_计划与预算" \
    "SRC2_计划预算"

move_tree_contents "$SRC2/7-集成整合/01-1 OCR" \
    "$DST/05_ePMS及关联系统资料/04_集成系统资料/01-1_OCR" \
    "SRC2_OCR"

move_tree_contents "$SRC2/7-集成整合/01-2 智能语音云平台" \
    "$DST/05_ePMS及关联系统资料/04_集成系统资料/01-2_智能语音云平台" \
    "SRC2_智能语音"

move_tree_contents "$SRC2/7-集成整合/01-3 绩效考核平台PEP" \
    "$DST/05_ePMS及关联系统资料/04_集成系统资料/01-3_绩效考核平台PEP" \
    "SRC2_PEP"

move_tree_contents "$SRC2/7-集成整合/02-1 SSO" \
    "$DST/05_ePMS及关联系统资料/04_集成系统资料/02-1_SSO" \
    "SRC2_SSO"

move_tree_contents "$SRC2/7-集成整合/02-2 生产经营管理系统WOMS" \
    "$DST/05_ePMS及关联系统资料/04_集成系统资料/02-2_生产经营管理系统WOMS" \
    "SRC2_WOMS"

move_tree_contents "$SRC2/7-集成整合/02-3 EIIS" \
    "$DST/05_ePMS及关联系统资料/04_集成系统资料/02-3_EIIS" \
    "SRC2_EIIS"

move_tree_contents "$SRC2/7-集成整合/03-1 CA" \
    "$DST/05_ePMS及关联系统资料/04_集成系统资料/03-1_CA" \
    "SRC2_CA"

move_tree_contents "$SRC2/7-集成整合/03-2 RPA机器人" \
    "$DST/05_ePMS及关联系统资料/04_集成系统资料/03-2_RPA机器人" \
    "SRC2_RPA"

move_tree_contents "$SRC2/7-集成整合/04-1 制度管理系统" \
    "$DST/05_ePMS及关联系统资料/04_集成系统资料/04-1_制度管理系统" \
    "SRC2_制度管理"

move_tree_contents "$SRC2/7-集成整合/04-2 协同办公" \
    "$DST/05_ePMS及关联系统资料/04_集成系统资料/04-2_协同办公" \
    "SRC2_协同办公"

move_tree_contents "$SRC2/7-集成整合/04-3 内部竞聘" \
    "$DST/05_ePMS及关联系统资料/04_集成系统资料/04-3_内部竞聘" \
    "SRC2_内部竞聘"

move_tree_contents "$SRC2/7-集成整合/05-1 技术标准" \
    "$DST/05_ePMS及关联系统资料/04_集成系统资料/05-1_技术标准" \
    "SRC2_技术标准系统"

move_tree_contents "$SRC2/7-集成整合/05-2 图纸管理" \
    "$DST/05_ePMS及关联系统资料/04_集成系统资料/05-2_图纸管理" \
    "SRC2_图纸管理"

move_tree_contents "$SRC2/7-集成整合/05-3 物联平台" \
    "$DST/05_ePMS及关联系统资料/04_集成系统资料/05-3_物联平台" \
    "SRC2_物联平台"

move_tree_contents "$SRC2/7-集成整合/06-1 ECN" \
    "$DST/05_ePMS及关联系统资料/04_集成系统资料/06-1_ECN" \
    "SRC2_ECN"

move_tree_contents "$SRC2/7-集成整合" \
    "$DST/05_ePMS及关联系统资料/03_集成整合调研结果" \
    "SRC2_集成整合剩余"

move_tree_contents "$SRC2/2-其他系统/工业互联网平台" \
    "$DST/05_ePMS及关联系统资料/05_工业互联网平台" \
    "SRC2_工业互联网"

move_tree_contents "$SRC2/2-其他系统/三峡梯调调度自动化系统顶层设计" \
    "$DST/05_ePMS及关联系统资料/06_三峡梯调调度自动化系统" \
    "SRC2_梯调系统"

move_tree_contents "$SRC2/10-长电2021信息化咨询" \
    "$DST/05_ePMS及关联系统资料/08_管理信息系统现状_信通院/长电2021信息化咨询" \
    "SRC2_2021信息化咨询"

move_tree_contents "$SRC2/2-其他系统" \
    "$DST/05_ePMS及关联系统资料/08_管理信息系统现状_信通院/其他系统剩余" \
    "SRC2_其他系统剩余"

# 06_数字底座与业务应用开发平台
move_tree_contents "$SRC2/3、EAM数字底座部分交付材料/20241230das交付物/09.1-运维实操赋能培训课件" \
    "$DST/06_数字底座与业务应用开发平台/03_运维实操培训课件" \
    "SRC2_运维培训"

move_tree_contents "$SRC2/3、EAM数字底座部分交付材料/20241230das交付物" \
    "$DST/06_数字底座与业务应用开发平台/01_数字化技术基础底座交付物/20241230das交付物" \
    "SRC2_数字底座20241230"

move_tree_contents "$SRC2/3、EAM数字底座部分交付材料/长江电力EAM业数字化业务应用开发平台验收材料" \
    "$DST/06_数字底座与业务应用开发平台/02_业务应用开发平台验收材料" \
    "SRC2_应用平台验收"

move_tree_contents "$SRC2/3、EAM数字底座部分交付材料" \
    "$DST/06_数字底座与业务应用开发平台/04_平台投标_实施方案_安全材料" \
    "SRC2_数字底座剩余"

# 08_制度标准与管理体系文件
move_tree_contents "$SRC2/3-生产管理体系文件（制度等）/0-管理制度/发电生产" \
    "$DST/08_制度标准与管理体系文件/02_公司级管理制度/发电生产" \
    "SRC2_制度_发电生产"

move_tree_contents "$SRC2/3-生产管理体系文件（制度等）/0-管理制度/投资管理" \
    "$DST/08_制度标准与管理体系文件/02_公司级管理制度/投资管理" \
    "SRC2_制度_投资管理"

move_tree_contents "$SRC2/3-生产管理体系文件（制度等）/0-管理制度/电能营销" \
    "$DST/08_制度标准与管理体系文件/02_公司级管理制度/电能营销" \
    "SRC2_制度_电能营销"

move_tree_contents "$SRC2/3-生产管理体系文件（制度等）/0-管理制度/经营管理" \
    "$DST/08_制度标准与管理体系文件/02_公司级管理制度/经营管理" \
    "SRC2_制度_经营管理"

move_tree_contents "$SRC2/3-生产管理体系文件（制度等）/0-管理制度/设施设备管理" \
    "$DST/08_制度标准与管理体系文件/02_公司级管理制度/设施设备管理" \
    "SRC2_制度_设施设备"

move_tree_contents "$SRC2/3-生产管理体系文件（制度等）/1-岗位说明书" \
    "$DST/08_制度标准与管理体系文件/03_岗位说明书" \
    "SRC2_岗位说明书"

move_tree_contents "$SRC2/3-生产管理体系文件（制度等）/2-下属单位/三峡电厂/现行制度及清单" \
    "$DST/08_制度标准与管理体系文件/04_三峡电厂制度与标准/现行制度及清单" \
    "SRC2_三峡制度"

move_tree_contents "$SRC2/3-生产管理体系文件（制度等）/2-下属单位/三峡电厂" \
    "$DST/08_制度标准与管理体系文件/04_三峡电厂制度与标准" \
    "SRC2_三峡电厂制度"

move_tree_contents "$SRC2/3-生产管理体系文件（制度等）/2-下属单位/溪洛渡电厂" \
    "$DST/08_制度标准与管理体系文件/05_溪洛渡电厂制度与标准" \
    "SRC2_溪洛渡制度"

move_tree_contents "$SRC2/3-生产管理体系文件（制度等）/2-下属单位/白鹤滩电厂" \
    "$DST/08_制度标准与管理体系文件/06_白鹤滩电厂制度与标准" \
    "SRC2_白鹤滩制度"

move_tree_contents "$SRC2/3-生产管理体系文件（制度等）" \
    "$DST/08_制度标准与管理体系文件/01_公司级管理体系文件" \
    "SRC2_生产管理体系剩余"

# 源目录2根目录散文件
move_root_files_by_rule_SRC2 "$SRC2"


# ============================================================
# 剩余未命中文件兜底
# ============================================================

move_remaining_files "$SRC1" \
    "$DST/99_重复_临时_待清理/07_来源不明文件/未命中_SRC1" \
    "SRC1_UNMATCHED"

move_remaining_files "$SRC2" \
    "$DST/99_重复_临时_待清理/07_来源不明文件/未命中_SRC2" \
    "SRC2_UNMATCHED"

log "========== 资料库迁移结束 =========="
log "日志文件：$LOG_FILE"

if [ "$DRY_RUN" = "--dry-run" ]; then
    log "当前为 DryRun 预演模式，未实际移动文件。"
else
    log "正式迁移完成。"
fi
```

---

# 二、执行方法

## 1. 赋予执行权限

```bash
chmod +x migrate_to_merged_tree.sh
```

## 2. 先预演，不实际移动

```bash
./migrate_to_merged_tree.sh \
"/home/你的用户名/文件夹1" \
"/home/你的用户名/文件夹2" \
"/home/你的用户名/长江电力生产经营管理数字化平台资料库_合并" \
--dry-run
```

## 3. 确认日志无误后正式迁移

```bash
./migrate_to_merged_tree.sh \
"/home/你的用户名/文件夹1" \
"/home/你的用户名/文件夹2" \
"/home/你的用户名/长江电力生产经营管理数字化平台资料库_合并"
```

---

# 三、迁移后建议检查

## 查看目标目录结构

```bash
tree "/home/你的用户名/长江电力生产经营管理数字化平台资料库_合并" -L 3
```

## 查看未命中文件

```bash
tree "/home/你的用户名/长江电力生产经营管理数字化平台资料库_合并/99_重复_临时_待清理/07_来源不明文件" -L 4
```

## 查看重复文件

```bash
tree "/home/你的用户名/长江电力生产经营管理数字化平台资料库_合并/99_重复_临时_待清理/01_疑似重复_待哈希确认" -L 4
```

## 查看迁移日志

```bash
ls -lh "/home/你的用户名/长江电力生产经营管理数字化平台资料库_合并/99_重复_临时_待清理/07_来源不明文件"/迁移日志_*.log
```

---

# 四、重要说明

这个脚本采用的是**规则迁移**，不是简单地把整个源目录原样搬过去。核心迁移逻辑如下：

| 源内容                            | 目标目录                |
| ------------------------------ | ------------------- |
| 早期规划、TGEAM前期咨询、招投标、合同、进度       | `01_项目管理_规划_合同_招投标` |
| 实施过程资料、交付模板、交付文档               | `02_项目实施与交付成果`      |
| 蓝图设计、业务架构、模型文件                 | `03_业务架构与蓝图设计工作区`   |
| 三峡调研、业务流程图、录音录像照片、IFS资料        | `04_调研与收资资料`        |
| ePMS、19系统整合、集成系统、工业互联网、梯调、历史系统 | `05_ePMS及关联系统资料`    |
| 数字底座、应用开发平台、UI规范、验收材料          | `06_数字底座与业务应用开发平台`  |
| 下阶段需求、数据质量、运行调度、精益生产           | `07_后续项目建设`         |
| 管理制度、三峡/溪洛渡/白鹤滩制度、公司背景         | `08_制度标准与管理体系文件`    |
| 外部方法论、数字化转型、EAM、SGERP、咨询案例     | `09_外部参考与方法论资料`     |
| 重复、临时、未命中、来源不明                 | `99_重复_临时_待清理`      |

特别注意：
`move_tree_contents "$SRC1/0、实施"` 这类兜底规则会把前面没有迁走的剩余文件放到对应的“剩余资料”目录，不会遗漏，但个别文件可能还需要迁移后人工二次整理。
