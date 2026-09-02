<template>
  <div class="pg">
    <!-- 页头 -->
    <div class="page-head">
      <div class="page-head-bar"></div>
      <span class="page-head-title">技能训练</span>
      <span class="page-head-sub">炼技Agent · AI伴学与智能评分</span>
      <button class="btn btn-primary" @click="autoFill">一键填写</button>
      <button class="btn-ghost" @click="restoreDraft">恢复草稿</button>
    </div>

    <!-- 任务头部 -->
    <div class="card task-head">
      <div class="th-l">
        <div class="th-title">
          <span class="th-name">桂三恒化工财务分析学习任务</span>
          <span class="tag tag-orange">中级任务</span>
          <span class="tag tag-green" v-if="result">已提交 · 炼技 Agent 已评分</span>
        </div>
        <div class="th-tags">
          <span class="kp">利润表认知</span>
          <span class="kp">盈利指标计算</span>
          <span class="kp">资产负债表认知</span>
          <span class="kp">偿债基础指标</span>
          <span class="kp">现金流量表认知</span>
          <span class="kp">现金流基础指标</span>
        </div>
      </div>
      <div class="th-r">
        <div class="th-meta">4 大模块 40 道选择题 · 在页面上逐题作答，提交后由炼技 Agent 自动评分并生成评语</div>
      </div>
    </div>

    <!-- 工作区 -->
    <div class="ws ws3">

      <!-- ===== 左侧：题单导航 + 提交 ===== -->
      <div class="nav-col">
        <div class="nav-card">
          <div class="nav-title">任务实施清单</div>
          <div class="nav-progress">
            完成度 <b>{{ donePct }}%</b>
            <span class="nav-progress-sub">{{ filledCount }} / {{ totalFields }} 项</span>
          </div>
          <div class="bar-bg tall"><div class="bar-fill f-blue" :style="{ width: donePct + '%' }"></div></div>
          <div class="nav-list">
            <div
              v-for="(n, i) in NAV" :key="n.id"
              class="nav-item" :class="{ on: curNav === i, done: navDone[i] }"
              @click="navGo(i)"
            >
              <span class="nav-dot">{{ navDone[i] ? '✓' : n.no }}</span>
              <span class="nav-name">{{ n.name }}</span>
            </div>
          </div>
          <button class="btn btn-primary btn-block mt8" :disabled="submitting" @click="submitWork">
            {{ submitting ? '正在提交评分…' : '提交作业' }}
          </button>
          <button class="btn btn-ghost btn-block" v-if="result" @click="goResult">查看 AI 评分</button>
          <button class="btn btn-ghost btn-block" @click="go('student-home')">返回任务地图</button>
          <div class="nav-foot">答案自动保存至本地 · 可随时刷新</div>
        </div>
      </div>

      <!-- ===== 右侧：答题区 ===== -->
      <div class="main-col">

        <!-- 0 学生信息 -->
        <div class="panel sh-panel" id="sec-info" style="--sc:#2B6CD6; --sc-bg:#EAF2FC;">
          <div class="step-head">
            <span class="sh-no">0</span>
            <div class="sh-info">
              <div class="sh-title">学生信息 <span class="sh-tag">作业封面</span></div>
              <div class="sh-desc">填写个人信息，将出现在作业 Markdown 封面中</div>
            </div>
            <span class="sh-pct">必填</span>
          </div>
          <div class="grid-4">
            <div class="field">
              <label>学生姓名 <b class="red">*</b></label>
              <input v-model="answers.name" placeholder="例：李明" />
            </div>
            <div class="field">
              <label>班级</label>
              <input v-model="answers.cls" placeholder="例：财务大数据分析班" />
            </div>
            <div class="field">
              <label>学号</label>
              <input v-model="answers.no" placeholder="例：2026FA031" />
            </div>
            <div class="field">
              <label>提交日期</label>
              <input v-model="answers.today" placeholder="2026-08-27" />
            </div>
          </div>
        </div>

        <!-- 1 任务一：中级选择题（40 题，含完整选项） -->
        <div class="panel sh-panel" id="sec-t1" style="--sc:#FA8C16; --sc-bg:#FFF3E8;">
          <div class="step-head">
            <span class="sh-no">1</span>
            <div class="sh-info">
              <div class="sh-title">任务一 · 财务知识入门测试 <span class="sh-tag">中级选择题 40</span></div>
              <div class="sh-desc">基于桂三恒化工 2024 年度财务报表作答，点击选项选择，答案按字母写入作业文档</div>
            </div>
            <button class="btn btn-primary btn-sm sh-dl" @click="downloadReport">📥 财务报表下载</button>
          </div>
          <div class="q-group" v-for="(g, gi) in T1_CHOICE" :key="'g' + gi">
            <div class="sub-title">{{ g.title }} <span class="muted">{{ g.items.length }} 题</span></div>
            <div class="q-block" v-for="(q, qi) in g.items" :key="'c' + gi + '-' + qi">
              <div class="q-title">{{ qStart(gi) + qi }}、{{ q.q }}</div>
              <div class="letter-chips">
                <button
                  v-for="(opt, oi) in q.opts" :key="LET[oi]"
                  class="lchip lchip-opt" :class="{ on: answers.choice[qStart(gi) + qi] === LET[oi] }"
                  @click="answers.choice[qStart(gi) + qi] = LET[oi]"
                ><span class="opt-letter">{{ LET[oi] }}</span><span class="opt-text">{{ opt }}</span></button>
              </div>
            </div>
          </div>
        </div>

        

        <!-- 2 任务一：简答与案例 -->
        <div class="panel sh-panel" id="sec-t1b" style="--sc:#722ED1; --sc-bg:#F7F0FF;">
          <div class="step-head">
            <span class="sh-no">2</span>
            <div class="sh-info">
              <div class="sh-title">任务一 · 简答与案例分析 <span class="sh-tag">3 简答 + 1 案例</span></div>
              <div class="sh-desc">结合桂三恒化工财务数据与财务分析基础知识作答</div>
            </div>
          </div>
          <div class="field">
            <label>简答 1 · 简述业财融合，并结合桂三恒化工说明财务如何支撑业务 <b class="red">*</b></label>
            <textarea v-model="answers.short.s1" placeholder="例：业财融合指财务与业务一体化运作。桂三恒化工收入、成本、存货等数据由业务系统产生，财务据此核算利润表；财务分析结果又反过来指导采购与销售定价、成本管控……"></textarea>
          </div>
          <div class="field">
            <label>简答 2 · 说明利润与现金的关系，并结合 2024 年数据说明</label>
            <textarea v-model="answers.short.s2" placeholder="例：净利润 2,067.50 万元、经营现金净流入 2,137.42 万元，二者接近但不完全一致——受应收账款、存货、折旧等非现金项目影响……"></textarea>
          </div>
          <div class="field">
            <label>简答 3 · 列举财务人员的职业底线</label>
            <textarea v-model="answers.short.s3" placeholder="例：不做假账、不提前确认收入、不挪用资金、保守商业秘密、坚持独立客观、遵守会计法规……"></textarea>
          </div>
          <div class="field">
            <label>案例分析 · 2024 年营业收入增长 6.16%，净利润却下降 68.32%，请分析主要原因</label>
            <textarea v-model="answers.short.case" placeholder="例：营业成本增速（+18.53%）显著高于营业收入增速，毛利率由 16.44% 降至 6.70%，费用增长叠加，导致净利润大幅下降……"></textarea>
          </div>
        </div>

        <!-- 3 任务二：报表勾稽 -->
        <div class="panel sh-panel" id="sec-t2a" style="--sc:#13A8A8; --sc-bg:#E6F7F7;">
          <div class="step-head">
            <span class="sh-no">3</span>
            <div class="sh-info">
              <div class="sh-title">任务二 · 三张报表勾稽验证 <span class="sh-tag">3 项验证</span></div>
              <div class="sh-desc">验证年报数据勾稽关系，并写出计算过程与结论（期初 / 本期 / 期末列为只读参考数据）</div>
            </div>
          </div>
          <div class="table-wrap">
            <table>
              <thead>
                <tr><th>验证项</th><th>期初</th><th>本期变动</th><th>期末</th><th>计算过程</th><th>学生结论</th></tr>
              </thead>
              <tbody>
                <tr v-for="(r, ri) in RECON_ROWS" :key="ri">
                  <td>{{ r.name }}</td>
                  <td class="num">{{ r.begin }}</td>
                  <td class="num">{{ r.chg }}</td>
                  <td class="num">{{ r.end }}</td>
                  <td><input class="in-cell" v-model="answers.recon[ri].process" placeholder="写出验证过程" /></td>
                  <td><input class="in-cell" v-model="answers.recon[ri].conclusion" placeholder="例：勾稽一致" /></td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <!-- 4 任务二：指标底稿 -->
        <div class="panel sh-panel" id="sec-t2b" style="--sc:#FA8C16; --sc-bg:#FFF3E8;">
          <div class="step-head">
            <span class="sh-no">4</span>
            <div class="sh-info">
              <div class="sh-title">任务二 · 财务指标树与计算工作底稿 <span class="sh-tag">8 项指标</span></div>
              <div class="sh-desc">按计算公式填写 2024 年度指标结果，并给出简要解读</div>
            </div>
          </div>
          <div class="table-wrap">
            <table>
              <thead>
                <tr><th>指标</th><th>计算公式</th><th>学生计算结果</th><th>学生解读</th></tr>
              </thead>
              <tbody>
                <tr v-for="(r, ri) in IDX_ROWS" :key="ri">
                  <td>{{ r.name }}</td>
                  <td class="formula">{{ r.formula }}</td>
                  <td><input class="in-cell" v-model="answers.idx[ri].res" :placeholder="r.ph" /></td>
                  <td><input class="in-cell" v-model="answers.idx[ri].note" placeholder="简要解读" /></td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <!-- 5 任务二：财务备忘录 -->
        <div class="panel sh-panel" id="sec-t2c" style="--sc:#2B6CD6; --sc-bg:#EAF2FC;">
          <div class="step-head">
            <span class="sh-no">5</span>
            <div class="sh-info">
              <div class="sh-title">任务二 · 2024 年度财务速览备忘录 <span class="sh-tag">报送财务总监</span></div>
              <div class="sh-desc">用一段话概括公司 2024 年度财务状况，并提出下一步工作安排</div>
            </div>
          </div>
          <div class="grid-2">
            <div class="field">
              <label>致</label>
              <input v-model="answers.memo.to" />
            </div>
            <div class="field">
              <label>发自</label>
              <input v-model="answers.memo.from" placeholder="例：林晓" />
            </div>
          </div>
          <div class="field">
            <label>主题</label>
            <input v-model="answers.memo.subject" />
          </div>
          <div class="field">
            <label>总体印象</label>
            <textarea v-model="answers.memo.impression" placeholder="例：2024 年公司营业收入保持增长，但成本增速较快、毛利率下滑，净利润大幅下降，盈利质量走弱；偿债能力尚可，现金流质量下降需要关注。"></textarea>
          </div>
          <div class="field">
            <label>下一步工作</label>
            <textarea v-model="answers.memo.next" placeholder="例：跟踪采购成本与期间费用管控；加强应收账款回款；持续关注经营现金流改善。"></textarea>
          </div>
        </div>

        <!-- 6 任务三：盈利能力 -->
        <div class="panel sh-panel" id="sec-t3a" style="--sc:#FF4D4F; --sc-bg:#FFECEC;">
          <div class="step-head">
            <span class="sh-no">6</span>
            <div class="sh-info">
              <div class="sh-title">任务三 · 盈利能力深度分析 <span class="sh-tag">同比 + 杜邦 ROE</span></div>
              <div class="sh-desc">对比 2024 与 2023 年数据，填写同比变动与判断</div>
            </div>
          </div>
          <div class="table-wrap">
            <table>
              <thead>
                <tr><th>项目</th><th>2024 年</th><th>2023 年</th><th>学生计算（同比%）</th><th>学生分析</th></tr>
              </thead>
              <tbody>
                <tr v-for="(r, ri) in PROF_ROWS" :key="ri">
                  <td>{{ r.name }}</td>
                  <td class="num">{{ r.y24 }}</td>
                  <td class="num">{{ r.y23 }}</td>
                  <td><input class="in-cell" v-model="answers.prof[ri].calc" :placeholder="r.ph" /></td>
                  <td><input class="in-cell" v-model="answers.prof[ri].note" placeholder="例：收入小幅增长" /></td>
                </tr>
              </tbody>
            </table>
          </div>
          <div class="dupont-row">
            <div class="dupont-item">
              <div class="dupont-k">杜邦 ROE <span class="muted">（2024 10.41% / 2023 36.89%）</span></div>
              <input class="in-cell" v-model="answers.dupont.calc" placeholder="学生计算/判断：例 受销售净利率下降影响" />
            </div>
            <div class="dupont-item">
              <div class="dupont-k">杜邦分析（学生解读）</div>
              <input class="in-cell" v-model="answers.dupont.note" placeholder="例：成本上升压低 ROE" />
            </div>
          </div>
          <div class="sub-title mt2">费用明细（参考数据）</div>
          <div class="table-wrap">
            <table>
              <thead><tr><th>费用项目</th><th>2024 年</th><th>2023 年</th></tr></thead>
              <tbody>
                <tr v-for="(r, ri) in PROF_FEES" :key="'f' + ri">
                  <td>{{ r.name }}</td><td class="num">{{ r.y24 }}</td><td class="num">{{ r.y23 }}</td>
                </tr>
              </tbody>
            </table>
          </div>
          <div class="field mt2">
            <label>改善建议</label>
            <textarea v-model="answers.advice" placeholder="例：控制原料采购成本；压缩管理费用；提高销售收入。"></textarea>
          </div>
        </div>

        <!-- 7 任务三：偿债与现金流 -->
        <div class="panel sh-panel" id="sec-t3b" style="--sc:#2B6CD6; --sc-bg:#EAF2FC;">
          <div class="step-head">
            <span class="sh-no">7</span>
            <div class="sh-info">
              <div class="sh-title">任务三 · 偿债能力、营运能力与现金流质量 <span class="sh-tag">5 指标 + 分析</span></div>
              <div class="sh-desc">计算指标并给出结论，最后撰写现金流质量分析</div>
            </div>
          </div>
          <div class="table-wrap">
            <table>
              <thead><tr><th>项目</th><th>学生计算</th><th>学生结论</th></tr></thead>
              <tbody>
                <tr v-for="(r, ri) in SOLV_ROWS" :key="ri">
                  <td>{{ r.name }}</td>
                  <td><input class="in-cell" v-model="answers.solv[ri].calc" :placeholder="r.ph" /></td>
                  <td><input class="in-cell" v-model="answers.solv[ri].conclusion" placeholder="例：偿债能力较好" /></td>
                </tr>
              </tbody>
            </table>
          </div>
          <div class="field mt2">
            <label>现金流质量分析</label>
            <textarea v-model="answers.cashText" placeholder="例：经营现金流为正，能够覆盖固定资产支出，但比去年下降较多，需加快回款、减少资金占用。"></textarea>
          </div>
        </div>

        <!-- 8 任务四：风险诊断与报告 -->
        <div class="panel sh-panel" id="sec-t4" style="--sc:#FA8C16; --sc-bg:#FFF3E8;">
          <div class="step-head">
            <span class="sh-no">8</span>
            <div class="sh-info">
              <div class="sh-title">任务四 · 风险诊断与风险洞察报告 <span class="sh-tag">3 风险 + 报告摘要</span></div>
              <div class="sh-desc">结合数据证据判断风险，给出原因分析与建议</div>
            </div>
          </div>
          <div class="risk-card" v-for="(r, ri) in RISK_ROWS" :key="ri">
            <div class="risk-head"><span class="risk-no">{{ r.id }}</span><span class="risk-ev">{{ r.ev }}</span></div>
            <div class="risk-fields">
              <input class="in-cell" v-model="answers.risk[ri].judge" placeholder="学生判断：例 盈利能力下降风险较大" />
              <input class="in-cell" v-model="answers.risk[ri].reason" placeholder="原因分析：例 原材料成本增加，费用增长" />
              <input class="in-cell" v-model="answers.risk[ri].suggest" placeholder="管理建议：例 控制采购成本，减少费用" />
            </div>
          </div>
          <div class="risk-fields-label">上：学生判断 · 中：原因分析 · 下：管理建议</div>
          <div class="sub-title mt2">报告摘要与汇报提纲</div>
          <div class="field">
            <label>摘要</label>
            <textarea v-model="answers.report.summary" placeholder="例：公司收入保持增长，但成本增长较快导致净利润下降；建议控制原材料采购成本、加强回款管理。"></textarea>
          </div>
          <div class="field">
            <label>短期行动</label>
            <textarea v-model="answers.report.action" placeholder="例：检查采购价格与费用，跟进应收账款。"></textarea>
          </div>
          <div class="field">
            <label>汇报提纲</label>
            <textarea v-model="answers.report.outline" placeholder="例：公司概况；经营情况；盈利分析；风险；建议。"></textarea>
          </div>
        </div>

        <!-- 9 任务五：职业伦理 -->
        <div class="panel sh-panel" id="sec-t5" style="--sc:#52C41A; --sc-bg:#F1FAE8;">
          <div class="step-head">
            <span class="sh-no">9</span>
            <div class="sh-info">
              <div class="sh-title">任务五 · 职业道德与数据合规 <span class="sh-tag">2 情境</span></div>
              <div class="sh-desc">结合职业道德与数据合规要求作答</div>
            </div>
          </div>
          <div class="field" v-for="(r, ri) in ETHIC_ROWS" :key="ri">
            <label>{{ r.t }} · {{ r.q }}</label>
            <textarea v-model="answers.ethic[ri].answer" placeholder="写明你的处理方式与理由"></textarea>
          </div>
        </div>

        <!-- 10 提交与评分结果 -->
        <div class="panel result-panel" id="sec-result" v-if="result" style="--sc:#2B6CD6; --sc-bg:#EAF2FC;">
          <div class="step-head">
            <span class="sh-no">✓</span>
            <div class="sh-info">
              <div class="sh-title">炼技 Agent 评分结果 <span class="sh-tag">Markdown 评语</span></div>
              <div class="sh-desc">提交时间：{{ submittedAt }} · 可重新作答后再次提交，评语将更新</div>
            </div>
          </div>
          <div class="md-body" v-html="mdToHtml(result)"></div>
          <div class="sc-actions">
            <button class="btn btn-primary btn-sm" @click="goResult">回到评分结果顶部</button>
            <button class="btn btn-ghost btn-sm" @click="showMd = !showMd">查看提交的作业 Markdown</button>
          </div>
          <div class="md-preview" v-if="showMd">
            <pre>{{ buildMarkdown() }}</pre>
          </div>
        </div>

        <div class="submit-row">
          <button class="btn btn-primary" :disabled="submitting" @click="submitWork">{{ submitting ? '正在提交评分…' : '提交作业' }}</button>
          <button class="btn btn-ghost" @click="go('student-home')">返回任务地图</button>
          <span class="submit-tip">提交后炼技 Agent 将读取作业文件并返回 Markdown 评语</span>
        </div>
      </div>

      <!-- ===== 右侧：AI 财务导师（接入另一对话智能体，流式答疑） ===== -->
      <div class="ai-col">
        <div class="ai-panel">
          <div class="mentor-head">
            <span class="mentor-avatar">AI</span>
            <div class="mentor-title">
              <b>AI 财务导师</b>
              <small>伴学答疑</small>
            </div>
            <button class="btn-ghost btn-sm" @click="resetMentor">清空</button>
          </div>
          <div class="mentor-chips">
            <button class="qchip" v-for="q in mentorQuestions" :key="q" @click="sendMentor(q)">{{ q }}</button>
          </div>
          <div class="mentor-hist">
            <div v-for="(m, i) in chatMsgs" :key="i" class="chat-msg" :class="{ user: m.role === 'user' }">
              <span class="chat-av">{{ m.role === 'user' ? '我' : 'AI' }}</span>
              <div class="chat-bubble" v-if="m.role === 'user'">{{ m.text }}</div>
              <div class="chat-bubble md-body" v-else v-html="mdToHtml(m.text)"></div>
            </div>
            <div v-if="chatting" class="chat-msg ai">
              <span class="chat-av">AI</span>
              <div class="chat-bubble typing">正在思考…</div>
            </div>
          </div>
          <div class="mentor-input">
            <input v-model="chatInput" placeholder="向导师提问，如：毛利率下降有哪些原因？" @keydown.enter="sendMentor()" :disabled="chatting" />
            <button class="btn btn-primary btn-sm" @click="sendMentor()" :disabled="chatting">发送</button>
          </div>
          <div class="mentor-foot" v-if="!mentorReady">导师接入中：请将 MNT_KEY 配置为财务导师智能体 API Key</div>
        </div>
      </div>
    </div>

    <!-- 提示 -->
    <div v-if="tip" class="tip">{{ tip }}</div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from "vue";

const emit = defineEmits(["navigate"]);

// ============ 数据服务（localStorage 共享，PRESET 兜底） ============
function lsGet(k) { try { const v = localStorage.getItem(k); return v ? JSON.parse(v) : null; } catch (e) { return null; } }
function lsSet(k, v) { try { localStorage.setItem(k, JSON.stringify(v)); } catch (e) { /* 忽略 */ } }
const LS_STU = "fd_student_v2";
const LS_ANS = "fd_train_answer_v1";

// ============ AI 智能体配置 ============
// 炼技 Agent（提交作业评分）：Dify 工作流接口（workflows/run）
const AI_URL = "https://agent.gjt-smart.com/v1/workflows/run";
const AI_KEY = "app-LQY5t0hJxu3kC6ViiDFBUKHm";
const AI_BASE = AI_URL.replace(/\/workflows\/run$/, "");
const UPLOAD_URL = AI_BASE + "/files/upload";
const AI_USER = "student-training";

// AI 财务导师（另一对话智能体 · 伴学答疑）：chat-messages 接口
const MNT_URL = "https://agent.gjt-smart.com/v1/chat-messages";
const MNT_KEY = "app-sRbxW1npTm5iQAbn92ki8ND5";
const MNT_USER = "student-mentor";
const mentorReady = MNT_KEY && String(MNT_KEY).indexOf("xxxx") < 0;

// ============ 静态数据：桂三恒化工题库（依据训练题库 Excel 11 个表） ============
const LET = ["A", "B", "C", "D"];

// 任务一：初级选择题（40 题，依据《财务大数据分析学生训练测试卷》逐题补全题干与 A/B/C/D 选项）
const T1_CHOICE = [
  {
    title: "任务1-1 · 认识利润表",
    items: [
      { q: "公司2024年度营业收入为（　）万元。", opts: ["53,210.16", "50,123.56", "5,321.02", "532.10"] },
      { q: "利润表从“营业收入”到“净利润”的正确计算路径是（　）。", opts: ["营业利润=营业收入-营业成本-税金及附加-期间费用-资产减值损失+公允价值变动损益+投资收益+其他收益等", "净利润=利润总额+所得税费用", "营业成本包含销售费用、管理费用和财务费用", "利润总额=净利润-营业外收支净额"] },
      { q: "公司2024年度营业成本约为（　）亿元。", opts: ["4.18", "4.96", "5.32", "6.70"] },
      { q: "关于三大报表之间的勾稽关系，下列表述正确的是（　）。", opts: ["资产负债表“未分配利润”期末余额应等于利润表“净利润”", "资产负债表“货币资金”期末余额应与现金流量表“期末现金及现金等价物余额”一致", "利润表“营业收入”应等于现金流量表“销售商品收到现金”", "三张报表之间不存在勾稽关系"] },
      { q: "化工企业利润表中反映行业特点的“特殊科目”通常不包括（　）。", opts: ["研发费用", "专项储备（安全生产费）", "存货跌价准备", "长期待摊费用摊销"] },
      { q: "公司2024年度期间费用（销售费用+管理费用+研发费用+财务费用）合计约为（　）万元。", opts: ["860", "1,026", "2,488", "8,598"] }
    ]
  },
  {
    title: "任务1-2 · 盈利指标计算",
    items: [
      { q: "公司2024年度销售毛利率为（　）。（毛利率=（营业收入-营业成本）/营业收入）", opts: ["6.70%", "16.44%", "3.89%", "5.04%"] },
      { q: "公司2024年度销售净利率为（　）。（销售净利率=净利润/营业收入）", opts: ["16.44%", "13.02%", "3.89%", "5.04%"] },
      { q: "公司毛利率由2023年的16.44%降至2024年的6.70%，最可能的原因是（　）。", opts: ["营业收入大幅下降", "营业成本增速显著高于营业收入增速", "财务费用激增", "税金及附加大幅增加"] },
      { q: "公司2024年度营业利润率为（　）。（营业利润率=营业利润/营业收入）", opts: ["3.89%", "5.04%", "6.70%", "11.01%"] },
      { q: "公司2024年度净利润为2,067.50万元，2023年度为6,527.58万元，则净利润同比（　）。", opts: ["增长68.33%", "下降68.33%", "增长6.16%", "基本持平"] },
      { q: "公司2024年度营业收入同比增长6.16%，而净利润下降68.33%，这种现象可概括为（　）。", opts: ["增收不增利", "增利不增收", "减收增利", "亏损扩大"] }
    ]
  },
  {
    title: "任务2-1 · 资产负债表认知",
    items: [
      { q: "公司2024年末资产总计为（　）万元。", opts: ["39,296.54", "40,243.95", "19,865.21", "20,378.74"] },
      { q: "下列对会计恒等式“资产=负债+所有者权益”的变形，正确的是（　）。", opts: ["负债=资产-所有者权益", "所有者权益=资产+负债", "资产=所有者权益-负债", "三者之间没有数量关系"] },
      { q: "下列属于公司非流动资产的是（　）。", opts: ["应收账款", "存货", "固定资产净额", "货币资金"] },
      { q: "判断流动资产与流动负债，通常以（　）为划分标准。", opts: ["1个月", "1年或一个营业周期", "3年", "5年"] },
      { q: "公司固定资产原价14,265.94万元、累计折旧6,108.39万元，固定资产净额8,157.55万元，这说明（　）。", opts: ["公司固定资产老化程度较高（累计折旧率约42.8%），属典型重资产企业", "公司为轻资产企业", "固定资产无折旧", "公司无形资产占比高"] },
      { q: "公司2024年末所有者权益合计为（　）万元。", opts: ["20,378.74", "19,865.21", "40,243.95", "17,693.21"] }
    ]
  },
  {
    title: "任务2-2 · 偿债基础指标计算",
    items: [
      { q: "公司2024年末流动比率为（　）。（流动比率=流动资产÷流动负债）", opts: ["1.87", "2.03", "0.50", "1.02"] },
      { q: "公司2024年末速动比率为（　）。（速动比率=（流动资产-存货）÷流动负债）", opts: ["1.87", "2.03", "3.02", "1.02"] },
      { q: "公司2024年末资产负债率为（　）。", opts: ["50.64%", "54.98%", "40.24%", "19.87%"] },
      { q: "一般认为，流动比率、速动比率、资产负债率的常见参考标准分别是（　）。", opts: ["2.0、1.0、40%—60%", "1.0、2.0、50%—70%", "2.0、2.0、20%—40%", "1.5、1.0、60%—80%"] },
      { q: "与年初相比，公司资产负债率由54.98%降至50.64%，说明（　）。", opts: ["长期偿债压力有所减轻、资本结构趋于稳健", "公司资不抵债", "短期偿债能力下降", "资本结构没有变化"] },
      { q: "若公司流动比率2.03、速动比率1.87，二者的差额主要由（　）引起。", opts: ["存货", "货币资金", "应收账款", "固定资产"] }
    ]
  },
  {
    title: "任务3-1 · 认识现金流量表",
    items: [
      { q: "公司2024年度经营活动产生的现金流量净额为（　）万元。", opts: ["2,137.42", "4,746.89", "4,683.67", "5,341.25"] },
      { q: "下列属于投资活动现金流量的是（　）。", opts: ["销售商品收到的现金", "购建固定资产支付的现金", "取得借款收到的现金", "支付给职工以及为职工支付的现金"] },
      { q: "公司2024年度投资活动产生的现金流量净额为-534.13万元，说明（　）。", opts: ["公司进行资本扩张，购建固定资产等支出大于投资收回", "公司投资活动取得净收益", "投资活动收支相抵", "公司停止对外投资"] },
      { q: "净利润为正但经营活动现金流量净额较低甚至为负，最不可能的原因是（　）。", opts: ["应收账款大幅增加", "存货积压占用资金", "大量赊销未收回现金", "销售商品大量收到现金"] },
      { q: "公司2024年度筹资活动产生的现金流量净额为（　）。", opts: ["30,803,673.98元", "-30,665,020.44元", "21,374,231.98元", "46,836,651.39元"] },
      { q: "公司2024年度“现金及现金等价物净增加额”为4,683.67万元，期末货币资金46,927,531.66元，二者勾稽关系为（　）。", opts: ["期末货币资金=期初货币资金+现金净增加额", "期末货币资金=现金净增加额", "期末货币资金=期初货币资金", "二者没有关系"] }
    ]
  },
  {
    title: "任务3-2 · 现金流基础指标计算",
    items: [
      { q: "公司2024年度经营活动现金流量净额与净利润之比为（　）。", opts: ["0.50", "1.03", "1.50", "2.03"] },
      { q: "公司2024年度自由现金流量（经营活动现金流量净额-购建固定资产支出）约为（　）万元。", opts: ["2,137", "1,603", "534", "4,684"] },
      { q: "通常认为，经营活动现金流量净额与净利润之比（　）说明盈利质量高。", opts: ["小于0.5", "介于0.5—1之间", "大于1", "等于0"] },
      { q: "公司2024年度经营活动现金流量净额占营业收入的比重约为（　），低于化工行业均值10%—15%。", opts: ["4.02%", "9.47%", "13.72%", "1.03%"] },
      { q: "公司2023年度经营活动现金流量净额与营业收入之比约为9.47%，2024年度约为4.02%，说明（　）。", opts: ["收现能力下降、现金流质量走弱", "收现能力增强", "盈利质量明显改善", "无变化"] },
      { q: "判断企业自我“造血”能力时，最关键的指标组合是（　）。", opts: ["经营活动现金流量净额持续为正、自由现金流为正", "投资活动现金流量净额为正", "净利润持续增长", "资产负债率持续下降"] }
    ]
  },
  {
    title: "任务4-1 · 财务速览备忘录",
    items: [
      { q: "三张报表中，反映企业“家底”（财务状况）的是（　）。", opts: ["利润表", "资产负债表", "现金流量表", "所有者权益变动表"] },
      { q: "反映企业“经营成果”的是（　）。", opts: ["利润表", "资产负债表", "现金流量表", "报表附注"] },
      { q: "反映企业“血液”（现金流动）的是（　）。", opts: ["利润表", "资产负债表", "现金流量表", "报表附注"] },
      { q: "综合公司2024年三大报表数据，下列表述最准确的是（　）。", opts: ["盈利能力强劲、偿债能力稳健、现金流充沛", "增收不增利、偿债能力尚可、现金流质量下降", "亏损严重、资不抵债", "无任何财务风险"] }
    ]
  }
];
// 题目在全部 7 组中的全局起始题号（第 1 组从 1 开始）
function qStart(gi) {
  let n = 0;
  for (let i = 0; i < gi; i++) n += T1_CHOICE[i].items.length;
  return n + 1;
}

// 任务二：报表勾稽（期初 / 本期变动 / 期末 为只读参考数据）
const RECON_ROWS = [
  { name: "货币资金勾稽", begin: "90,880.27", chg: "46,836,651.39", end: "46,927,531.66" },
  { name: "净利润与未分配利润", begin: "109,340,754.10", chg: "20,675,006.58", end: "130,015,760.68" },
  { name: "经营活动现金流（间接法）", begin: "净利润 20,675,006.58", chg: "折旧、财务费用、营运资本变动", end: "21,374,231.98" }
];

// 任务二：指标工作底稿
const IDX_ROWS = [
  { name: "流动比率", formula: "流动资产 ÷ 流动负债", ph: "例 2.03" },
  { name: "速动比率", formula: "(流动资产 - 存货) ÷ 流动负债", ph: "例 1.87" },
  { name: "资产负债率", formula: "负债总额 ÷ 资产总额", ph: "例 50.64%" },
  { name: "销售净利率", formula: "净利润 ÷ 营业收入", ph: "例 3.89%" },
  { name: "毛利率", formula: "(营业收入 - 营业成本) ÷ 营业收入", ph: "例 6.70%" },
  { name: "ROE（净资产收益率）", formula: "净利润 ÷ 所有者权益", ph: "例 10.41%" },
  { name: "应收账款周转率", formula: "营业收入 ÷ 平均应收账款", ph: "例 5.14 次" },
  { name: "经营现金流 / 净利润", formula: "经营活动现金流净额 ÷ 净利润", ph: "例 1.03" }
];

// 任务三：盈利能力（2024 / 2023 为只读参考数据）
const PROF_ROWS = [
  { name: "营业收入", y24: "532,101,648.88", y23: "501,235,641.13", ph: "例 +6.16%" },
  { name: "营业成本", y24: "496,461,589.47", y23: "418,840,819.62", ph: "例 +18.53%" },
  { name: "销售费用", y24: "2,487,984.30", y23: "2,089,431.40", ph: "例 0.47%" },
  { name: "管理费用", y24: "2,795,790.44", y23: "2,625,571.96", ph: "例 0.53%" },
  { name: "研发费用", y24: "890,464.11", y23: "506,480.25", ph: "例 0.17%" }
];
const PROF_FEES = [
  { name: "营业成本", y24: "496,461,589.47", y23: "418,840,819.62" },
  { name: "销售费用", y24: "2,487,984.30", y23: "2,089,431.40" },
  { name: "管理费用", y24: "2,795,790.44", y23: "2,625,571.96" },
  { name: "研发费用", y24: "890,464.11", y23: "506,480.25" }
];

// 任务三：偿债能力、营运能力与现金流质量
const SOLV_ROWS = [
  { name: "利息保障倍数", ph: "例 11.44" },
  { name: "Z-score（破产风险）", ph: "例 3.10" },
  { name: "应收账款周转率", ph: "例 5.14 次" },
  { name: "应收账款周转天数", ph: "例 71 天" },
  { name: "经营现金流 / 净利润", ph: "例 1.03" }
];

// 任务四：风险诊断（数据证据为只读提示）
const RISK_ROWS = [
  { id: "R-01", ev: "收入 +6.16%、成本 +18.53%、净利润 -68.32%" },
  { id: "R-04", ev: "应收账款 109,870,857.32 元（金额较大）" },
  { id: "R-05", ev: "经营现金流 21,374,231.98 元，同比下降" }
];

// 任务五：职业伦理情境
const ETHIC_ROWS = [
  { t: "情境 1 · 提前确认收入", q: "管理层要求提前确认收入以完成业绩目标，你作为财务人员应如何处理？" },
  { t: "情境 2 · 数据保密", q: "工作中会接触客户信息、成本构成、报价策略等敏感数据，应如何处理以保证数据合规与保密？" }
];

// ============ 左侧题单导航 ============
const NAV = [
  { no: 1, name: "学生信息", id: "sec-info" },
  { no: 2, name: "任务一 · 知识入门测试", id: "sec-t1" },
  { no: 3, name: "任务一 · 简答与案例", id: "sec-t1b" },
  { no: 4, name: "任务二 · 报表勾稽", id: "sec-t2a" },
  { no: 5, name: "任务二 · 指标底稿", id: "sec-t2b" },
  { no: 6, name: "任务二 · 财务备忘录", id: "sec-t2c" },
  { no: 7, name: "任务三 · 盈利能力", id: "sec-t3a" },
  { no: 8, name: "任务三 · 偿债与现金流", id: "sec-t3b" },
  { no: 9, name: "任务四 · 风险诊断与报告", id: "sec-t4" },
  { no: 10, name: "任务五 · 职业伦理", id: "sec-t5" }
];

// ============ 工具函数 ============
function todayStr() {
  const d = new Date();
  return d.getFullYear() + "-" + String(d.getMonth() + 1).padStart(2, "0") + "-" + String(d.getDate()).padStart(2, "0");
}
function orNA(v) {
  const s = (v === null || v === undefined) ? "" : String(v);
  return s.trim() ? s.trim() : "（未作答）";
}
function arrN(n, fill) { const a = []; for (let i = 0; i < n; i++) a.push(JSON.parse(JSON.stringify(fill))); return a; }
const EMPTY_RECON = { process: "", conclusion: "" };
const EMPTY_IDX = { res: "", note: "" };
const EMPTY_PROF = { calc: "", note: "" };
const EMPTY_SOLV = { calc: "", conclusion: "" };
const EMPTY_RISK = { judge: "", reason: "", suggest: "" };
const EMPTY_ETHIC = { answer: "" };

// ============ 学生信息（从学习端 student 中读取姓名/班级，PRESET 兜底） ============
function loadStudent() {
  try {
    const raw = lsGet(LS_STU);
    return (raw && raw.name) ? raw : { name: "林晓", cls: "财务2433" };
  } catch (e) { return { name: "林晓", cls: "财务2433" }; }
}
function defaultAnswers() {
  const st = loadStudent();
  return {
    name: st.name || "", cls: st.cls || "", no: "2026FA031", today: todayStr(),
    choice: {},          // 选择题答案 {1:'A',2:'C',...}（题号 1-40）
    short: { s1: "", s2: "", s3: "", case: "" },
    recon: arrN(RECON_ROWS.length, EMPTY_RECON),
    idx: arrN(IDX_ROWS.length, EMPTY_IDX),
    memo: { to: "财务总监", from: "", subject: "2024年度财务状况初步分析", impression: "", next: "" },
    prof: arrN(PROF_ROWS.length, EMPTY_PROF),
    dupont: { calc: "", note: "" },
    advice: "",
    solv: arrN(SOLV_ROWS.length, EMPTY_SOLV),
    cashText: "",
    risk: arrN(RISK_ROWS.length, EMPTY_RISK),
    report: { summary: "", action: "", outline: "" },
    ethic: arrN(ETHIC_ROWS.length, EMPTY_ETHIC)
  };
}
function sanitizeAns(s) {
  if (!s || typeof s !== "object") return defaultAnswers();
  const def = defaultAnswers();
  const pick = function (k, d) { const v = s[k]; return v === undefined || v === null ? d : v; };
  const mapArr = function (k, d, n) {
    const v = pick(k, []);
    const out = [];
    for (let i = 0; i < n; i++) out.push(Object.assign({}, d, v[i] || {}));
    return out;
  };
  return {
    name: String(pick("name", def.name) || ""), cls: String(pick("cls", def.cls) || ""),
    no: String(pick("no", def.no) || ""), today: String(pick("today", def.today) || def.today),
    choice: pick("choice", {}),
    short: Object.assign({}, def.short, pick("short", {})),
    recon: mapArr("recon", EMPTY_RECON, RECON_ROWS.length),
    idx: mapArr("idx", EMPTY_IDX, IDX_ROWS.length),
    memo: Object.assign({}, def.memo, pick("memo", {})),
    prof: mapArr("prof", EMPTY_PROF, PROF_ROWS.length),
    dupont: Object.assign({}, def.dupont, pick("dupont", {})),
    advice: String(pick("advice", "") || ""),
    solv: mapArr("solv", EMPTY_SOLV, SOLV_ROWS.length),
    cashText: String(pick("cashText", "") || ""),
    risk: mapArr("risk", EMPTY_RISK, RISK_ROWS.length),
    report: Object.assign({}, def.report, pick("report", {})),
    ethic: mapArr("ethic", EMPTY_ETHIC, ETHIC_ROWS.length)
  };
}

// ============ 页面状态 ============
const answers = ref(sanitizeAns(lsGet(LS_ANS)));
const tip = ref("");
const submitting = ref(false);
const result = ref("");
const submittedAt = ref("");
const showMd = ref(false);
const curNav = ref(0);
let tipTimer = null;
let io = null;

function showTip(t) { tip.value = t; clearTimeout(tipTimer); tipTimer = setTimeout(function () { tip.value = ""; }, 2600); }
function restoreDraft() { answers.value = sanitizeAns(lsGet(LS_ANS)); showTip("已恢复本地最新草稿"); }

// ============ 财务报表下载（内嵌官方《财务报表》文档，点击下载原始 docx 文件） ============
const REPORT_B64 =
    "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";
function downloadReport() {
  try {
    const bin = atob(REPORT_B64);
    const bytes = new Uint8Array(bin.length);
    for (let i = 0; i < bin.length; i++) bytes[i] = bin.charCodeAt(i);
    const blob = new Blob([bytes], { type: "application/vnd.openxmlformats-officedocument.wordprocessingml.document" });
    const a = document.createElement("a");
    a.href = URL.createObjectURL(blob);
    a.download = "财务报表.docx";
    document.body.appendChild(a);
    a.click();
    a.remove();
    setTimeout(function () { URL.revokeObjectURL(a.href); }, 1000);
    showTip("财务报表已下载（桂三恒化工 2024 年度财务报表）");
  } catch (e) {
    showTip("下载失败，请稍后重试");
  }
}

// ============ 一键填写（读取《财务大数据训练题库_模拟学生作业_中等水平.md》演示答案） ============
function autoFill() {
  const a = answers.value;
  // 学生信息
  a.name = "陈晓宇";
  a.cls = loadStudent().cls || "财务2433";
  a.no = "2026FD102";
  a.today = "2026-08-25";
  // 选择题 40 题（组序：任务1-1/1-2、任务2-1/2-2、任务3-1/3-2、任务4-1，与 md 各任务 1-6 题一一对应）
  const CH = ["A","A","C","B","C","A","A","C","B","A","B","A","B","A","C","B","B","B","B","A","A","A","B","A","A","B","A","D","B","A","B","B","C","A","A","B","B","A","C","A"];
  CH.forEach(function (v, i) { a.choice[i + 1] = v; });
  // 任务一 · 简答与案例分析
  a.short.s1 = "业财融合强调财务与业务一体化运作。桂三恒化工的收入、成本、库存等数据由业务系统产生，财务据此完成核算与报表；财务分析结论（如毛利率下滑、应收周转偏慢）又反过来指导采购议价、信用政策与费用控制，实现财务对业务的支撑。";
  a.short.s2 = "净利润 2,067.50 万元对应经营活动现金净流入 2,137.42 万元，二者接近（比值 1.03），说明利润基本有现金保障；差异主要来自存货增加、经营性应收应付变动以及折旧等非现金项目。";
  a.short.s3 = "不做假账、不提前确认收入、不挪用资金、保守商业秘密、坚持独立客观、遵守会计准则与财经法规。";
  a.short.case = "化工行业原材料价格波动会影响企业成本，市场竞争也可能使产品提价困难。桂三恒收入增长 6.16%，但营业成本增长 18.53%，毛利率从 16.44% 降到 6.70%，是净利润下降 68.32% 的主要原因。";
  // 任务二 · 报表勾稽
  a.recon[0].process = "期末货币资金 = 期初货币资金 + 本期现金净增加额 = 90,880.27 + 46,836,651.39 = 46,927,531.66 元";
  a.recon[0].conclusion = "勾稽一致";
  a.recon[1].process = "未分配利润期末 = 期初 109,340,754.10 + 本期净利润 20,675,006.58 = 130,015,760.68 元";
  a.recon[1].conclusion = "勾稽一致";
  a.recon[2].process = "以净利润 20,675,006.58 元为起点，加回折旧、财务费用并调整存货（-11,594,261.89）、经营性应收（+23,629,360.31）、经营性应付（-23,269,592.14）等变动，得经营活动现金流净额 21,374,231.98 元";
  a.recon[2].conclusion = "与现金流量表经营活动现金流净额一致";
  // 任务二 · 指标底稿
  a.idx[0].res = "2.03";     a.idx[0].note = "高于参考值 2.0，短期偿债能力较好";
  a.idx[1].res = "1.87";     a.idx[1].note = "高于参考值 1.0，流动资产变现后可覆盖流动负债";
  a.idx[2].res = "50.64%";   a.idx[2].note = "处于 40%—60% 常见区间，资本结构较为稳健";
  a.idx[3].res = "3.89%";    a.idx[3].note = "较 2023 年 13.02% 明显下降，盈利水平走弱";
  a.idx[4].res = "6.70%";    a.idx[4].note = "较 2023 年 16.44% 大幅下降，成本上升是主因";
  a.idx[5].res = "10.41%";   a.idx[5].note = "较 2023 年 36.89% 大幅回落";
  a.idx[6].res = "5.14 次";  a.idx[6].note = "回款速度偏慢，周转天数约 71 天";
  a.idx[7].res = "1.03";     a.idx[7].note = "利润基本有现金保障";
  // 任务二 · 财务备忘录
  a.memo.to = "财务总监";
  a.memo.from = "陈晓宇";
  a.memo.subject = "2024 年度财务状况初步分析";
  a.memo.impression = "2024 年公司营业收入保持增长（+6.16%），但营业成本增速较快（+18.53%），毛利率由 16.44% 降至 6.70%，净利润大幅下降，盈利能力风险较高；偿债能力总体尚可（资产负债率 50.64%、Z 值 3.07）；经营现金流虽为正但同比下降、占收入仅 4.02%，现金流质量一般。";
  a.memo.next = "控制原材料采购成本；加快应收账款催收；控制期间费用；建立月度现金流预警表。";
  // 任务三 · 盈利能力
  a.prof[0].calc = "+6.16%";   a.prof[0].note = "收入小幅增长";
  a.prof[1].calc = "+18.53%";  a.prof[1].note = "成本增速显著高于收入，是利润下滑主因";
  a.prof[2].calc = "+19.07%";  a.prof[2].note = "销售费用增速较快，需关注投入产出";
  a.prof[3].calc = "+6.48%";   a.prof[3].note = "管理费用小幅增长，总体平稳";
  a.prof[4].calc = "+75.81%";  a.prof[4].note = "研发投入增加明显";
  a.dupont.calc = "按连环替代法分解：销售净利率影响约 -25.88 个百分点，总资产周转率影响约 +0.40 个百分点，权益乘数影响约 -1.00 个百分点，合计约 -26.48 个百分点";
  a.dupont.note = "ROE 由 36.89% 降至 10.41%，主因是销售净利率下降（成本上升、利润率下滑），资产使用效率未明显变差";
  a.advice = "1. 与主要原料供应商签订长期采购协议，降低采购价格波动；2. 对利润较低的产品重新核算成本，减少低毛利订单；3. 控制管理费用，定期检查费用报销和非必要支出。";
  // 任务三 · 偿债与现金流
  a.solv[0].calc = "11.44";    a.solv[0].conclusion = "利息保障倍数充分，偿还利息压力较小";
  a.solv[1].calc = "3.07";     a.solv[1].conclusion = "Z 值大于 2.99，处于安全区，可正常获得授信";
  a.solv[2].calc = "5.14 次";  a.solv[2].conclusion = "回款速度偏慢，应收账款占用资金较多";
  a.solv[3].calc = "71 天";    a.solv[3].conclusion = "高于行业约 45 天，需加强客户信用管理与催收";
  a.solv[4].calc = "1.03";     a.solv[4].conclusion = "利润基本有现金保障，盈利质量尚可";
  a.cashText = "企业经营现金流为正，自由现金流也为正（约 1,603.30 万元），不是严重失血。但经营现金流同比下降较多，应收账款周转天数约 71 天、高于行业水平，说明回款占用了现金；经营现金流占收入仅 4.02%，低于行业 10%—15%。综合判断企业造血能力为“一般”。";
  // 任务四 · 风险诊断
  a.risk[0].judge = "盈利能力下降风险较大";
  a.risk[0].reason = "营业成本增长 18.53% 快于收入增长 6.16%，毛利率降至 6.70%，净利润下降 68.32%";
  a.risk[0].suggest = "控制原材料采购成本，压缩期间费用，提升产品毛利";
  a.risk[1].judge = "应收账款回收风险较高";
  a.risk[1].reason = "应收账款约 1.10 亿元、金额较大，周转天数 71 天高于行业 45 天";
  a.risk[1].suggest = "按客户信用分级管理，加强逾期催收，加快回款";
  a.risk[2].judge = "现金流质量下降风险中等";
  a.risk[2].reason = "经营现金流虽为正但同比下降，占收入仅 4.02%，低于行业 10%—15%";
  a.risk[2].suggest = "建立月度现金流预警，压缩库存与资金占用";
  // 任务四 · 报告
  a.report.summary = "公司营业收入增长 6.16%，但营业成本增长 18.53%，毛利率由 16.44% 降至 6.70%，净利润大幅下降，增收不增利；偿债能力总体尚可（资产负债率 50.64%、Z 值 3.07）；现金流质量一般（经营现金流占收入 4.02%）。综合财务健康度 72 分、B 级，主要风险为盈利能力下降与应收账款回收。";
  a.report.action = "控制原材料采购成本；加快应收账款催收；控制期间费用；建立月度现金流预警表。";
  a.report.outline = "执行摘要、背景、盈利能力、偿债能力、现金流、风险判断、管理建议。";
  // 任务五 · 职业伦理
  a.ethic[0].answer = "拒绝管理层要求，严格按收入确认原则，不得为完成业绩目标提前确认收入；向管理层说明提前确认收入会造成虚假记载与合规风险，建议按实际履约进度确认；如仍被强制要求，应向上级或合规部门报告并保留书面证据。";
  a.ethic[1].answer = "遵守保密义务，对客户信息、成本构成、报价策略等敏感数据按最小授权原则访问，不在非授权场合透露、不通过个人设备随意转发；使用数据符合合规要求，离职或交接时按规定脱敏归档。";
  showTip("已一键填写《中等水平模拟作业》答案，检查后可提交");
  window.scrollTo({ top: 0, behavior: "smooth" });
}
watch(answers, function (v) { lsSet(LS_ANS, v); }, { deep: true });

// ============ 完成度统计 ============
function fillCount(arr, keys) {
  let n = 0;
  for (let i = 0; i < arr.length; i++) {
    const o = arr[i] || {};
    for (let k = 0; k < keys.length; k++) { if (String(o[keys[k]] || "").trim()) n++; }
  }
  return n;
}
function sectionFilled(i) {
  const a = answers.value;
  switch (i) {
    case 0: return (String(a.name).trim() ? 1 : 0) + (String(a.no).trim() ? 1 : 0);
    case 1: return Object.keys(a.choice).length;
    case 2: return [a.short.s1, a.short.s2, a.short.s3, a.short.case].filter(function (x) { return String(x).trim(); }).length;
    case 3: return fillCount(a.recon, ["process", "conclusion"]);
    case 4: return fillCount(a.idx, ["res", "note"]);
    case 5: return [a.memo.from, a.memo.impression, a.memo.next].filter(function (x) { return String(x).trim(); }).length;
    case 6: return fillCount(a.prof, ["calc", "note"]) + (String(a.dupont.calc).trim() ? 1 : 0) + (String(a.dupont.note).trim() ? 1 : 0) + (String(a.advice).trim() ? 1 : 0);
    case 7: return fillCount(a.solv, ["calc", "conclusion"]) + (String(a.cashText).trim() ? 1 : 0);
    case 8: return fillCount(a.risk, ["judge", "reason", "suggest"]) + [a.report.summary, a.report.action, a.report.outline].filter(function (x) { return String(x).trim(); }).length;
    case 9: return fillCount(a.ethic, ["answer"]);
    default: return 0;
  }
}
const SECTION_MAX = [2, 40, 4, 6, 16, 3, 13, 11, 12, 2];
const totalFields = SECTION_MAX.reduce(function (p, c) { return p + c; }, 0);
const filledCount = computed(function () {
  let n = 0;
  SECTION_MAX.forEach(function (m, i) { n += Math.min(m, sectionFilled(i)); });
  return n;
});
const donePct = computed(function () {
  let sum = 0;
  SECTION_MAX.forEach(function (m, i) { sum += Math.min(1, sectionFilled(i) / m); });
  return Math.min(100, Math.round(sum / SECTION_MAX.length * 100));
});
const navDone = computed(function () {
  return SECTION_MAX.map(function (m, i) { return sectionFilled(i) / m >= 0.5; });
});

// ============ 导航 ============
function navGo(i) {
  curNav.value = i;
  const el = document.getElementById(NAV[i].id);
  if (el) el.scrollIntoView({ behavior: "smooth", block: "start" });
}
function goResult() {
  const el = document.getElementById("sec-result");
  if (el) el.scrollIntoView({ behavior: "smooth", block: "start" });
}

// ============ Markdown 组装（参照训练题库 Markdown 示例格式） ============
function mdChoice() {
  const lines = [];
  let no = 1;
  T1_CHOICE.forEach(function (g) {
    g.items.forEach(function () {
      lines.push(no + ". " + (answers.value.choice[no] || "（未作答）"));
      no++;
    });
  });
  return lines.join("\n");
}
function buildMarkdown() {
  const a = answers.value;
  const L = [];
  L.push("# 财务大数据分析学生训练作业");
  L.push("");
  L.push("学生：" + orNA(a.name));
  L.push("班级：" + orNA(a.cls));
  L.push("学号：" + orNA(a.no));
  L.push("提交日期：" + orNA(a.today));
  L.push("");
  L.push("---");
  L.push("");
  // 任务一
  L.push("## 一、任务一：财务知识入门测试");
  L.push("");
  L.push("### 1. 初级选择题（40 题）");
  L.push("");
  L.push(mdChoice());
  L.push("");
  L.push("### 2. 简答与案例分析");
  L.push("");
  L.push("**简答1　简述业财融合并结合桂三恒说明**");
  L.push("");
  L.push(orNA(a.short.s1));
  L.push("");
  L.push("**简答2　说明利润与现金的关系**");
  L.push("");
  L.push(orNA(a.short.s2));
  L.push("");
  L.push("**简答3　列举职业底线**");
  L.push("");
  L.push(orNA(a.short.s3));
  L.push("");
  L.push("**案例分析　收入增长6.16%、净利润下降68.32%的原因**");
  L.push("");
  L.push(orNA(a.short.case));
  L.push("");
  // 任务二
  L.push("---");
  L.push("");
  L.push("## 二、任务二：三张报表勾稽与指标计算");
  L.push("");
  L.push("### 1. 报表勾稽验证");
  L.push("");
  L.push("| 验证项 | 期初 | 本期变动 | 期末 | 计算过程 | 学生结论 |");
  L.push("|---|---|---|---|---|---|");
  RECON_ROWS.forEach(function (r, i) {
    L.push("| " + r.name + " | " + r.begin + " | " + r.chg + " | " + r.end + " | " + orNA(a.recon[i].process) + " | " + orNA(a.recon[i].conclusion) + " |");
  });
  L.push("");
  L.push("### 2. 财务指标树与计算工作底稿");
  L.push("");
  L.push("| 指标 | 计算公式 | 学生计算结果 | 学生解读 |");
  L.push("|---|---|---|---|");
  IDX_ROWS.forEach(function (r, i) {
    L.push("| " + r.name + " | " + r.formula + " | " + orNA(a.idx[i].res) + " | " + orNA(a.idx[i].note) + " |");
  });
  L.push("");
  L.push("### 3. 2024年度财务速览备忘录");
  L.push("");
  L.push("致：" + orNA(a.memo.to));
  L.push("");
  L.push("发自：" + orNA(a.memo.from));
  L.push("");
  L.push("主题：" + orNA(a.memo.subject));
  L.push("");
  L.push("总体印象：" + orNA(a.memo.impression));
  L.push("");
  L.push("下一步工作：" + orNA(a.memo.next));
  L.push("");
  // 任务三
  L.push("---");
  L.push("");
  L.push("## 三、任务三：盈利能力、偿债能力与营运能力分析");
  L.push("");
  L.push("### 1. 盈利能力深度分析");
  L.push("");
  L.push("| 项目 | 2024年 | 2023年 | 学生计算 | 学生分析 |");
  L.push("|---|---:|---:|---|---|");
  PROF_ROWS.forEach(function (r, i) {
    L.push("| " + r.name + " | " + r.y24 + " | " + r.y23 + " | " + orNA(a.prof[i].calc) + " | " + orNA(a.prof[i].note) + " |");
  });
  L.push("");
  L.push("杜邦 ROE：2024 年 10.41% / 2023 年 36.89%，学生计算/判断：" + orNA(a.dupont.calc) + "，学生解读：" + orNA(a.dupont.note));
  L.push("");
  L.push("改善建议：" + orNA(a.advice));
  L.push("");
  L.push("### 2. 偿债能力、营运能力与现金流质量");
  L.push("");
  L.push("| 项目 | 学生计算 | 学生结论 |");
  L.push("|---|---|---|");
  SOLV_ROWS.forEach(function (r, i) {
    L.push("| " + r.name + " | " + orNA(a.solv[i].calc) + " | " + orNA(a.solv[i].conclusion) + " |");
  });
  L.push("");
  L.push("现金流质量分析：" + orNA(a.cashText));
  L.push("");
  // 任务四
  L.push("---");
  L.push("");
  L.push("## 四、任务四：风险诊断与风险洞察报告");
  L.push("");
  L.push("### 1. 风险诊断与证据链");
  L.push("");
  L.push("| 风险编号 | 数据证据 | 学生判断 | 原因分析 | 管理建议 |");
  L.push("|---|---|---|---|---|");
  RISK_ROWS.forEach(function (r, i) {
    L.push("| " + r.id + " | " + r.ev + " | " + orNA(a.risk[i].judge) + " | " + orNA(a.risk[i].reason) + " | " + orNA(a.risk[i].suggest) + " |");
  });
  L.push("");
  L.push("### 2. 风险洞察报告摘要与汇报提纲");
  L.push("");
  L.push("摘要：" + orNA(a.report.summary));
  L.push("");
  L.push("短期行动：" + orNA(a.report.action));
  L.push("");
  L.push("汇报提纲：" + orNA(a.report.outline));
  L.push("");
  // 任务五
  L.push("---");
  L.push("");
  L.push("## 五、任务五：职业道德与数据合规");
  L.push("");
  ETHIC_ROWS.forEach(function (r, i) {
    L.push("### " + r.t);
    L.push("");
    L.push("问题描述：" + r.q);
    L.push("");
    L.push("学生作答：" + orNA(a.ethic[i].answer));
    L.push("");
  });
  return L.join("\n");
}

// ============ 轻量 Markdown 渲染（安全转义，用于展示 Agent 评语） ============
function esc(s) {
  return String(s).replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;");
}
function inline(s) {
  return esc(s)
    .replace(/\*\*([^*]+)\*\*/g, "<b>$1</b>")
    .replace(/`([^`]+)`/g, "<code>$1</code>");
}
function isTableSep(l) {
  return l.indexOf("-") >= 0 && /^\s*\|?[\s|:-]+\|?\s*$/.test(l);
}
function mdToHtml(md) {
  const lines = String(md || "").split(/\r?\n/);
  const out = [];
  let i = 0;
  while (i < lines.length) {
    const t = lines[i].trim();
    if (!t) { i++; continue; }
    // 表格
    if (t.indexOf("|") >= 0 && !isTableSep(t)) {
      const head = [];
      const body = [];
      while (i < lines.length && lines[i].trim().indexOf("|") >= 0) {
        const rl = lines[i].trim();
        if (isTableSep(rl)) { i++; continue; }
        const cells = rl.replace(/^\|/, "").replace(/\|$/, "").split("|").map(function (c) { return inline(c.trim()); });
        if (!head.length) head.push(cells); else body.push(cells);
        i++;
      }
      if (head.length) {
        let h = '<div class="md-tbl"><table><thead><tr>' + head[0].map(function (c) { return "<th>" + c + "</th>"; }).join("") + "</tr></thead><tbody>";
        h += body.map(function (r) { return "<tr>" + r.map(function (c) { return "<td>" + c + "</td>"; }).join("") + "</tr>"; }).join("");
        out.push(h + "</tbody></table></div>");
      }
      continue;
    }
    // 标题
    const hm = t.match(/^(#{1,4})\s+(.*)$/);
    if (hm) {
      out.push('<div class="md-h md-h' + hm[1].length + '">' + inline(hm[2]) + "</div>");
      i++; continue;
    }
    // 分割线
    if (/^(-{3,}|\*{3,})$/.test(t)) { out.push('<hr class="md-hr" />'); i++; continue; }
    // 无序列表
    if (/^[-*•]\s+/.test(t)) {
      const items = [];
      while (i < lines.length && /^[-*•]\s+/.test(lines[i].trim())) { items.push(inline(lines[i].trim().replace(/^[-*•]\s+/, ""))); i++; }
      out.push('<ul class="md-ul">' + items.map(function (x) { return "<li>" + x + "</li>"; }).join("") + "</ul>");
      continue;
    }
    // 有序列表
    if (/^\d+[.、]\s+/.test(t)) {
      const items = [];
      while (i < lines.length && /^\d+[.、]\s+/.test(lines[i].trim())) { items.push(inline(lines[i].trim().replace(/^\d+[.、]\s+/, ""))); i++; }
      out.push('<ol class="md-ol">' + items.map(function (x) { return "<li>" + x + "</li>"; }).join("") + "</ol>");
      continue;
    }
    // 引用
    if (/^>\s?/.test(t)) { out.push('<div class="md-quote">' + inline(t.replace(/^>\s?/, "")) + "</div>"); i++; continue; }
    // 段落
    out.push('<p class="md-p">' + inline(t) + "</p>");
    i++;
  }
  return out.join("");
}
function mdToPlain(md) {
  return mdToHtml(md).replace(/<[^>]+>/g, "");
}

// ============ 提交作业（组装 Markdown → 文件上传 → 炼技 Agent 评分） ============
function buildWorkFile() {
  const md = buildMarkdown();
  return new File([md], "财务大数据分析训练作业_" + (String(answers.value.name).trim() || "学生") + ".md", {
    type: "text/markdown",
    lastModified: Date.now()
  });
}
async function uploadFile(file) {
  const fd = new FormData();
  fd.append("file", file);
  fd.append("user", AI_USER);
  const resp = await fetch(UPLOAD_URL, {
    method: "POST",
    headers: { "Authorization": "Bearer " + AI_KEY },
    body: fd
  });
  if (!resp.ok) throw new Error("作业文件上传失败（HTTP " + resp.status + "）");
  const data = await resp.json();
  const fid = data && (data.id || data.upload_file_id);
  if (!fid) throw new Error("上传未返回文件 ID，请稍后重试");
  return fid;
}
async function submitWork() {
  if (submitting.value) return;
  const a = answers.value;
  if (!String(a.name).trim()) { showTip("请先填写学生姓名"); navGo(0); return; }
  const chN = Object.keys(a.choice).length;
  const totalN = T1_CHOICE.reduce(function (p, g) { return p + g.items.length; }, 0);
  if (chN < totalN) {
    showTip("请完成任务一全部选择题后再提交（已完成 " + chN + "/" + totalN + "）");
    navGo(1);
    return;
  }
  submitting.value = true;
  try {
    const uploadFileId = await uploadFile(buildWorkFile());
    const resp = await fetch(AI_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json", "Authorization": "Bearer " + AI_KEY },
      body: JSON.stringify({
        inputs: {
          student_file: [{ type: "document", transfer_method: "local_file", upload_file_id: uploadFileId }]
        },
        response_mode: "blocking",
        user: AI_USER
      })
    });
    if (!resp.ok) throw new Error("炼技 Agent 评分失败（HTTP " + resp.status + "）");
    const data = await resp.json();
    // 工作流 blocking 响应：评分文本位于 data.outputs.text（兼容不同返回层级）
    const raw =
      (data && data.data && data.data.outputs && data.data.outputs.text) ||
      (data && data.outputs && data.outputs.text) ||
      (data && data.answer) ||
      "";
    if (!raw.trim()) throw new Error("炼技 Agent 未返回评语，请稍后重试");
    result.value = raw;
    submittedAt.value = new Date().toLocaleString("zh-CN", { hour12: false });
    showMd.value = false;
    showTip("提交成功，炼技 Agent 已完成评分");
    setTimeout(function () { goResult(); }, 120);
  } catch (err) {
    showTip(err && err.message ? err.message : "提交失败，请检查网络后重试");
  } finally {
    submitting.value = false;
  }
}

// ============ 平台页面跳转（无刷新，等同点击平台 Tab/菜单） ============
const MENU_OF = {
  "student-home": "学习主页", "student-task": "技能训练", "student-ethics": "立德润心", "student-feedback": "学习反馈",
  "teacher-tasks": "任务管理", "teacher-analytics": "学情分析", "teacher-review": "评分复核"
};
const URL_OF = {
  "student-home": "https://1040053947.yxd.ep.caih.com/apps/desktop/sapp/app_5ex10jl1o4/sapp_qguqtkatxm/box/form/form_udp0sjx0td/4328371521401607?menuId=4328371521401607&resource_type=FORM",
  "student-task": "https://1040053947.yxd.ep.caih.com/apps/desktop/sapp/app_5ex10jl1o4/sapp_qguqtkatxm/box/form/form_ix7gbehpyi/0859277446244543?menuId=0859277446244543&resource_type=FORM"
};
function clickMenu(menuName) {
  const boxes = document.querySelectorAll('.inner-tabbar-list .tab-box, .tab-box');
  for (let i = 0; i < boxes.length; i++) { if ((boxes[i].textContent || "").indexOf(menuName) >= 0) { boxes[i].click(); return true; } }
  const items = document.querySelectorAll('.menu-box [class*="menu-item"], [class*="menu-item"]');
  for (let i = 0; i < items.length; i++) { if ((items[i].textContent || "").indexOf(menuName) >= 0) { items[i].click(); return true; } }
  return false;
}
function go(view) {
  const menuName = MENU_OF[view];
  if (menuName && clickMenu(menuName)) return;
  const url = URL_OF[view];
  if (url) {
    try {
      const app = document.querySelector("#app");
      const vueApp = app && app.__vue_app__;
      const router = vueApp && vueApp.config && vueApp.config.globalProperties && vueApp.config.globalProperties.$router;
      if (router && typeof router.push === "function") {
        const tgt = new URL(url, location.origin);
        let path = tgt.pathname;
        try {
          const base = router.options && router.options.history && router.options.history.base;
          if (base && base !== "/" && path.indexOf(base) === 0) { path = path.slice(base.length); if (path.charAt(0) !== "/") path = "/" + path; }
        } catch (e2) { /* 读不到 base 则原样传入 */ }
        router.push(path + tgt.search);
        return;
      }
    } catch (e) { /* 忽略，继续下一级 */ }
    try { if (window.navigation && typeof window.navigation.navigate === "function") { window.navigation.navigate(url); return; } } catch (e) { /* 忽略 */ }
    window.location.href = url;
    return;
  }
  emit("navigate", view);
}

onMounted(function () {
  try {
    io = new IntersectionObserver(function (entries) {
      entries.forEach(function (e) {
        if (e.isIntersecting) {
          const idx = NAV.findIndex(function (n) { return n.id === e.target.id; });
          if (idx >= 0) curNav.value = idx;
        }
      });
    }, { rootMargin: "-15% 0px -75% 0px" });
    NAV.forEach(function (n) {
      const el = document.getElementById(n.id);
      if (el) io.observe(el);
    });
  } catch (e) { /* 环境不支持时忽略 */ }
});

// ============ AI 财务导师（另一对话智能体 · 流式答疑） ============
const mentorWelcome = "你好，我是你的 AI 财务分析导师。正在完成《桂三恒化工财务分析学习任务》中遇到指标含义、公式计算、分析思路或职业伦理问题，都可以问我，我会先引导你思考。";
const mentorQuestions = ["流动比率怎么算、怎么看？", "毛利率下降可能有哪些原因？", "杜邦分析怎么拆？", "利润和现金什么关系？"];
const chatMsgs = ref([{ role: "ai", text: mentorWelcome }]);
const chatInput = ref("");
const chatting = ref(false);
let mentorConv = ""; // conversation_id，保持上下文连贯

function resetMentor() {
  chatMsgs.value = [{ role: "ai", text: mentorWelcome }];
  mentorConv = "";
}
async function sendMentor(q) {
  const t = (q || chatInput.value || "").trim();
  if (!t || chatting.value) return;
  chatInput.value = "";
  chatMsgs.value.push({ role: "user", text: t });
  if (!mentorReady) {
    // 占位回复：导师智能体未配置
    chatting.value = true;
    setTimeout(function () {
      chatMsgs.value.push({ role: "ai", text: "（占位回复）AI 财务导师对话功能待接入。请在页面脚本中把 MNT_KEY 配置为财务导师智能体的 API Key。" });
      chatting.value = false;
    }, 400);
    return;
  }
  chatting.value = true;
  try {
    const resp = await fetch(MNT_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json", "Authorization": "Bearer " + MNT_KEY },
      body: JSON.stringify({
        inputs: { student_name: String(answers.value.name || "").trim(), task_name: "桂三恒化工财务分析学习任务作业" },
        response_mode: "streaming",
        conversation_id: mentorConv,
        query: t,
        user: MNT_USER,
        files: []
      })
    });
    if (!resp.ok || !resp.body) throw new Error("导师服务响应异常（HTTP " + resp.status + "）");
    const reader = resp.body.getReader();
    const decoder = new TextDecoder("utf-8");
    let buf = "";
    let asst = { role: "ai", text: "" };
    chatMsgs.value.push(asst);
    for (;;) {
      const { done, value } = await reader.read();
      if (done) break;
      buf += decoder.decode(value, { stream: true });
      let nl;
      while ((nl = buf.indexOf("\n")) >= 0) {
        const line = buf.slice(0, nl).replace(/\r$/, "");
        buf = buf.slice(nl + 1);
        if (!line || line.indexOf("data:") !== 0) continue;
        const payload = line.slice(5).trim();
        if (!payload) continue;
        try {
          const ev = JSON.parse(payload);
          // 兼容标准 Chat App（message）与 Agent Chat App（agent_message）两类事件
          if ((ev.event === "message" || ev.event === "agent_message") && ev.answer) { asst.text += ev.answer; chatMsgs.value = chatMsgs.value.slice(); }
          if ((ev.event === "message_end" || ev.event === "agent_message") && ev.conversation_id) mentorConv = ev.conversation_id || mentorConv;
        } catch (e2) { /* 忽略解析失败的行 */ }
      }
    }
    if (!asst.text.trim()) asst.text = "（导师未返回内容，请稍后重试）";
    scrollMentor();
  } catch (err) {
    chatMsgs.value.push({ role: "ai", text: "（导师连接失败：" + (err && err.message ? err.message : "请检查网络") + "）" });
  } finally {
    chatting.value = false;
  }
}
function scrollMentor() {
  const box = document.querySelector(".mentor-hist");
  if (box) box.scrollTop = box.scrollHeight;
}
</script>

<style scoped>
/* ============ 基线（对齐学习主页 / Ant 风格） ============ */
.pg { max-width: 1200px; margin: 0 auto; padding: 18px 16px 40px; }
.page-head { display: flex; align-items: center; gap: 10px; margin-bottom: 16px; flex-wrap: wrap; }
.page-head-bar { width: 4px; height: 22px; border-radius: 2px; background: #1677FF; }
.page-head-title { font-size: 20px; font-weight: 600; color: #1F2733; }
.page-head-sub { font-size: 13px; color: #97A1B2; flex: 1; }
.btn { display: inline-flex; align-items: center; justify-content: center; gap: 6px; height: 34px; padding: 0 16px; border-radius: 6px; font-size: 14px; font-weight: 600; border: 1px solid transparent; cursor: pointer; transition: all .15s ease; white-space: nowrap; }
.btn:disabled { opacity: .5; cursor: not-allowed; }
.btn-primary { background: #2B6CD6; color: #fff; }
.btn-primary:hover:not(:disabled) { background: #4D8BE8; }
.btn-ghost { color: #2B6CD6; background: #EAF2FC; }
.btn-ghost:hover { background: #DCEBFB; }
.btn-sm { height: 28px; padding: 0 12px; font-size: 13px; }
.btn-block { width: 100%; }
.mt { margin-top: 16px; }
.mt2 { margin-top: 20px; }
.mt8 { margin-top: 8px; }
.muted { color: #5A6577; font-size: 13px; }
.red { color: #FF4D4F; }
.card { background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 16px; box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05); }
.tag { display: inline-flex; align-items: center; font-size: 12px; padding: 1px 8px; border-radius: 4px; font-weight: 500; white-space: nowrap; }
.tag-blue { background: #EAF2FC; color: #1D4FA8; }
.tag-orange { background: #FFF3E8; color: #C96A06; }
.tag-red { background: #FFECEC; color: #D93025; }
.tag-green { background: #F1FAE8; color: #3C8E12; }
.bar-bg { background: #EEF3FA; border-radius: 99px; overflow: hidden; }
.bar-bg.tall { height: 8px; }
.bar-fill { height: 100%; border-radius: 99px; transition: width .4s ease; }
.f-blue { background: #2B6CD6; }
.tip { margin-top: 12px; background: #EAF2FC; border: 1px solid #DCEBFB; color: #1D4FA8; font-size: 13px; border-radius: 6px; padding: 8px 12px; }
@keyframes fadeIn { from { opacity: 0; transform: translateY(6px); } to { opacity: 1; transform: none; } }

/* ============ 任务头部 ============ */
.task-head { display: flex; flex-wrap: wrap; gap: 14px; align-items: flex-start; justify-content: space-between; margin-bottom: 16px; }
.th-l { flex: 1; min-width: 240px; }
.th-title { display: flex; align-items: center; gap: 8px; flex-wrap: wrap; margin-bottom: 10px; }
.th-name { font-size: 18px; font-weight: 600; color: #1F2733; }
.th-tags { display: flex; flex-wrap: wrap; gap: 6px; align-items: center; }
.th-r { flex: 1; min-width: 300px; }
.th-meta { font-size: 12px; color: #5A6577; text-align: right; line-height: 1.6; }
.kp { font-size: 12px; background: #fff; border: 1px solid #D6E3F5; color: #1D4FA8; padding: 2px 9px; border-radius: 99px; }

/* ============ 工作区 ============ */
.ws { display: grid; grid-template-columns: 1fr 370px; gap: 16px; align-items: start; }
.ws3 { grid-template-columns: 232px 1fr 330px; }
.main-col { min-width: 0; }

/* ============ 左侧导航 ============ */
.nav-col { position: sticky; top: 8px; }
.nav-card {
  background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 14px 12px;
  box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05);
  display: flex; flex-direction: column; gap: 8px;
}
.nav-title { font-size: 14px; font-weight: 700; color: #1F2733; }
.nav-progress { font-size: 12px; color: #5A6577; display: flex; align-items: baseline; gap: 6px; }
.nav-progress b { font-size: 16px; color: #2B6CD6; }
.nav-progress-sub { font-size: 11px; color: #97A1B2; margin-left: auto; }
.nav-list { display: flex; flex-direction: column; gap: 3px; margin-top: 2px; max-height: 430px; overflow-y: auto; }
.nav-item {
  display: flex; align-items: center; gap: 8px; padding: 7px 8px;
  border-radius: 6px; cursor: pointer; font-size: 12.5px; color: #5A6577;
  transition: all .15s ease;
}
.nav-item:hover { background: #F7FAFD; color: #1F2733; }
.nav-item.on { background: #EAF2FC; color: #1D4FA8; font-weight: 600; }
.nav-item.done:not(.on) { color: #52C41A; }
.nav-dot {
  width: 20px; height: 20px; border-radius: 50%; flex-shrink: 0;
  background: #EEF3FA; color: #97A1B2;
  display: flex; align-items: center; justify-content: center;
  font-size: 11px; font-weight: 700;
}
.nav-item.on .nav-dot { background: #2B6CD6; color: #fff; }
.nav-item.done .nav-dot { background: #52C41A; color: #fff; }
.nav-name { line-height: 1.35; }
.nav-foot { font-size: 11px; color: #97A1B2; text-align: center; }

/* ============ 面板 ============ */
.panel {
  background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 18px;
  box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05); margin-bottom: 16px; scroll-margin-top: 10px;
}
.sh-panel { border-top: 3px solid var(--sc); }
.step-head {
  display: flex; align-items: center; gap: 12px;
  padding-bottom: 13px; margin-bottom: 14px;
  border-bottom: 1px dashed #E3E9F2;
}
.sh-no {
  width: 32px; height: 32px; border-radius: 10px; flex-shrink: 0;
  background: var(--sc); color: #fff;
  display: flex; align-items: center; justify-content: center;
  font-size: 15px; font-weight: 700;
  box-shadow: 0 2px 8px rgba(16, 38, 76, .14);
}
.sh-info { min-width: 0; }
.sh-title { font-size: 15px; font-weight: 600; color: #1F2733; display: flex; align-items: center; gap: 8px; flex-wrap: wrap; }
.sh-title .sh-tag { font-size: 11px; font-weight: 600; color: var(--sc); background: var(--sc-bg); border-radius: 99px; padding: 2px 9px; }
.sh-desc { font-size: 12px; color: #5A6577; margin-top: 3px; }
.sh-pct { margin-left: auto; font-size: 12px; font-weight: 600; color: var(--sc); background: var(--sc-bg); padding: 4px 11px; border-radius: 99px; white-space: nowrap; flex-shrink: 0; }
.sh-dl { margin-left: auto; flex-shrink: 0; align-self: center; }

/* 表单 */
.field { margin-bottom: 14px; }
.field label { display: block; font-size: 13px; font-weight: 600; margin-bottom: 6px; line-height: 1.5; }
.field input, .field select, .field textarea {
  width: 100%; height: 34px; border: 1px solid #E3E9F2; border-radius: 6px; padding: 0 11px;
  background: #fff; color: #1F2733; transition: border-color .15s, box-shadow .15s;
}
.field textarea { height: auto; padding: 8px 11px; resize: vertical; min-height: 72px; line-height: 1.6; }
.field input:focus, .field select:focus, .field textarea:focus { outline: none; border-color: #2B6CD6; box-shadow: 0 0 0 2px #EAF2FC; }

/* ============ 网格布局 ============ */
.grid-4 { display: grid; grid-template-columns: repeat(4, 1fr); gap: 12px; }
.grid-3 { display: grid; grid-template-columns: repeat(3, 1fr); gap: 12px; }
.grid-2 { display: grid; grid-template-columns: repeat(2, 1fr); gap: 12px; }

/* ============ 子标题 / 题目块 ============ */
.sub-title { font-size: 13.5px; font-weight: 700; color: #1F2733; margin: 6px 0 10px; display: flex; align-items: center; gap: 8px; }
.sub-title::before { content: ""; width: 3px; height: 14px; border-radius: 2px; background: #2B6CD6; }
.q-block { background: #FAFBFD; border: 1px solid #EEF2F8; border-radius: 8px; padding: 12px 14px; margin-bottom: 12px; }
.q-title { font-size: 13px; color: #1F2733; line-height: 1.6; margin-bottom: 8px; }
.letter-chips { display: flex; flex-wrap: wrap; gap: 8px; }
.lchip {
  min-width: 34px; height: 30px; padding: 0 10px; border-radius: 7px;
  border: 1px solid #E3E9F2; background: #fff; color: #5A6577;
  font-size: 13px; font-weight: 600; cursor: pointer; transition: all .15s ease;
  display: inline-flex; align-items: center; justify-content: center;
}
.lchip:hover { border-color: #2B6CD6; color: #2B6CD6; }
.lchip.on { background: #2B6CD6; border-color: #2B6CD6; color: #fff; }
.lchip-sym { font-size: 15px; }
/* 带完整选项文字的作答按钮 */
.q-group { margin-bottom: 18px; }
.q-group:last-child { margin-bottom: 0; }
.lchip-opt {
  min-width: 0; height: auto; min-height: 32px; padding: 5px 11px;
  justify-content: flex-start; text-align: left; gap: 7px; flex: 0 1 auto;
}
.lchip-opt .opt-letter {
  width: 18px; height: 18px; border-radius: 50%; flex-shrink: 0;
  background: #EEF3FA; color: #5A6577; font-size: 11px; font-weight: 700;
  display: inline-flex; align-items: center; justify-content: center;
}
.lchip-opt.on .opt-letter { background: #fff; color: #2B6CD6; }
.lchip-opt .opt-text { font-weight: 400; line-height: 1.5; }

/* ============ 表格 / 底稿 ============ */
.table-wrap { overflow-x: auto; border: 1px solid #EEF2F8; border-radius: 8px; margin-bottom: 12px; background: #fff; }
.table-wrap table { width: 100%; border-collapse: collapse; font-size: 12.5px; min-width: 560px; }
.table-wrap th, .table-wrap td { border: 1px solid #EEF2F8; padding: 8px 10px; text-align: left; line-height: 1.5; vertical-align: top; }
.table-wrap th { background: #F7FAFD; color: #1F2733; font-weight: 600; white-space: nowrap; }
.table-wrap td.num { font-variant-numeric: tabular-nums; white-space: nowrap; }
.table-wrap td.formula { color: #1D4FA8; font-size: 12px; }
.in-cell {
  width: 100%; min-width: 120px; height: 30px; padding: 0 8px;
  border: 1px solid #E3E9F2; border-radius: 5px; font-size: 12.5px; background: #fff; color: #1F2733;
  transition: border-color .15s, box-shadow .15s;
}
.in-cell:focus { outline: none; border-color: #2B6CD6; box-shadow: 0 0 0 2px #EAF2FC; }

/* ============ 杜邦分析 ============ */
.dupont-row { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; margin-top: 10px; }
.dupont-item { background: #F7FAFD; border: 1px dashed #D6E3F5; border-radius: 8px; padding: 10px 12px; }
.dupont-k { font-size: 13px; font-weight: 600; color: #1F2733; margin-bottom: 8px; }

/* ============ 风险诊断卡片 ============ */
.risk-card { background: #FAFBFD; border: 1px solid #EEF2F8; border-radius: 8px; padding: 12px 14px; margin-bottom: 12px; }
.risk-head { display: flex; align-items: center; gap: 10px; margin-bottom: 10px; }
.risk-no {
  width: 24px; height: 24px; border-radius: 50%; flex-shrink: 0; background: #FF4D4F; color: #fff;
  font-size: 12px; font-weight: 700; display: flex; align-items: center; justify-content: center;
}
.risk-ev { font-size: 13px; font-weight: 600; color: #1F2733; line-height: 1.5; }
.risk-fields { display: flex; flex-direction: column; gap: 8px; }
.risk-fields-label { font-size: 11px; color: #97A1B2; text-align: right; margin-top: 2px; }

/* ============ AI 财务导师 · 右侧边栏 ============ */
.ai-col { position: sticky; top: 8px; }
.ai-panel {
  background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 16px;
  box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05);
  display: flex; flex-direction: column; gap: 10px;
}
.mentor-head { display: flex; align-items: center; gap: 9px; padding-bottom: 10px; border-bottom: 1px solid #E3E9F2; }
.mentor-avatar {
  width: 30px; height: 30px; border-radius: 9px; flex-shrink: 0;
  background: linear-gradient(135deg, #2B6CD6, #5B9BF0); color: #fff;
  font-size: 12px; font-weight: 700; display: flex; align-items: center; justify-content: center;
}
.mentor-title { flex: 1; min-width: 0; }
.mentor-title b { font-size: 14px; display: block; line-height: 1.3; }
.mentor-title small { font-size: 11px; color: #97A1B2; }
.mentor-chips { display: flex; flex-wrap: wrap; gap: 6px; }
.qchip {
  font-size: 12px; border: 1px solid #E3E9F2; background: #fff; color: #5A6577;
  padding: 4px 10px; border-radius: 99px; cursor: pointer; transition: .15s;
}
.qchip:hover { border-color: #2B6CD6; color: #2B6CD6; background: #EAF2FC; }
.mentor-hist { display: flex; flex-direction: column; gap: 10px; max-height: 340px; overflow-y: auto; padding: 4px 2px; }
.chat-msg { display: flex; gap: 8px; align-items: flex-start; }
.chat-msg.user { flex-direction: row-reverse; }
.chat-av {
  width: 26px; height: 26px; border-radius: 50%; flex-shrink: 0;
  background: linear-gradient(135deg, #2B6CD6, #5B9BF0); color: #fff;
  display: flex; align-items: center; justify-content: center; font-size: 11px; font-weight: 600;
}
.chat-msg.user .chat-av { background: linear-gradient(135deg, #FA8C16, #FFB066); }
.chat-bubble {
  max-width: 82%; background: #F7FAFD; border: 1px solid #EEF2F8;
  padding: 8px 12px; border-radius: 10px; font-size: 13px; line-height: 1.7; word-break: break-word; white-space: pre-wrap;
}
.chat-msg.user .chat-bubble { background: #2B6CD6; border-color: #2B6CD6; color: #fff; }
.chat-bubble.typing { color: #97A1B2; font-style: italic; animation: blink 1.2s ease-in-out infinite; }
@keyframes blink { 0%, 100% { opacity: .4; } 50% { opacity: 1; } }
.mentor-input { display: flex; gap: 8px; padding-top: 12px; border-top: 1px solid #E3E9F2; }
.mentor-input input {
  flex: 1; height: 32px; border: 1px solid #E3E9F2; border-radius: 6px; padding: 0 12px; font-size: 13px;
}
.mentor-input input:focus { outline: none; border-color: #2B6CD6; box-shadow: 0 0 0 2px #EAF2FC; }
.mentor-foot { font-size: 11px; color: #97A1B2; text-align: center; }

/* ============ 评分结果（Markdown 评语） ============ */
.result-panel { border-top: 3px solid var(--sc); }
.md-body { font-size: 13.5px; line-height: 1.75; color: #1F2733; }
.md-body :deep(.md-h) { font-weight: 700; margin: 14px 0 6px; color: #1D4FA8; }
.md-body :deep(.md-h1) { font-size: 17px; border-bottom: 1px solid #EEF2F8; padding-bottom: 6px; }
.md-body :deep(.md-h2) { font-size: 15px; }
.md-body :deep(.md-h3) { font-size: 14px; }
.md-body :deep(.md-h4) { font-size: 13.5px; color: #1F2733; }
.md-body :deep(.md-p) { margin: 6px 0; }
.md-body :deep(.md-ul), .md-body :deep(.md-ol) { margin: 6px 0; padding-left: 20px; }
.md-body :deep(.md-ul li), .md-body :deep(.md-ol li) { margin: 3px 0; }
.md-body :deep(.md-quote) { border-left: 3px solid #2B6CD6; background: #F7FAFD; padding: 6px 12px; margin: 8px 0; border-radius: 0 6px 6px 0; color: #5A6577; }
.md-body :deep(.md-hr) { border: none; border-top: 1px dashed #E3E9F2; margin: 14px 0; }
.md-body :deep(.md-tbl) { overflow-x: auto; margin: 10px 0; }
.md-body :deep(.md-tbl table) { width: 100%; border-collapse: collapse; font-size: 12.5px; border: 1px solid #EEF2F8; border-radius: 8px; }
.md-body :deep(.md-tbl th), .md-body :deep(.md-tbl td) { border: 1px solid #EEF2F8; padding: 6px 10px; text-align: left; }
.md-body :deep(.md-tbl th) { background: #F7FAFD; font-weight: 600; }
.md-body :deep(b) { color: #1D4FA8; }
.md-body :deep(code) { background: #EEF3FA; color: #C96A06; padding: 1px 5px; border-radius: 4px; font-size: 12px; }
.sc-actions { display: flex; flex-wrap: wrap; gap: 10px; margin-top: 14px; padding-top: 14px; border-top: 1px dashed #E3E9F2; }
.md-preview {
  margin-top: 14px; background: #FAFBFD; border: 1px solid #EEF2F8; border-radius: 8px;
  padding: 14px; font-size: 12.5px; line-height: 1.7; max-height: 380px; overflow: auto;
  white-space: pre-wrap; font-family: "SFMono-Regular", Consolas, monospace; color: #5A6577;
}

/* ============ 提交行 ============ */
.submit-row {
  display: flex; flex-wrap: wrap; align-items: center; gap: 12px; padding: 16px;
  background: #fff; border: 1px solid #E3E9F2; border-radius: 8px;
  box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05);
}
.submit-tip { font-size: 12px; color: #97A1B2; }

/* ============ 响应式 ============ */
@media (max-width: 1024px) {
  .ws3 { grid-template-columns: 1fr; }
  .nav-col, .ai-col { position: static; }
}
@media (max-width: 900px) {
  .ws, .ws3 { grid-template-columns: 1fr; }
  .grid-4 { grid-template-columns: repeat(2, 1fr); }
  .grid-3, .grid-2, .dupont-row { grid-template-columns: 1fr; }
  .submit-row { flex-direction: column; align-items: stretch; }
  .submit-tip { text-align: center; }
}
</style>