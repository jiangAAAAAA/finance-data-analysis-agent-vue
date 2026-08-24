<template>
  <div class="pg">
    <!-- 页头 -->
    <div class="page-head">
      <div class="page-head-bar"></div>
      <span class="page-head-title">任务工作台</span>
      <span class="page-head-sub">财数智析 · 炼技 Agent 伴学任务执行</span>
      <button class="btn-ghost" @click="refresh">刷新数据</button>
    </div>

    <!-- 任务头部 -->
    <div class="card task-head">
      <div class="th-l">
        <div class="th-title">
          <span class="th-name">{{ task.name }}</span>
          <span class="tag" :class="lvTag(task.level)">{{ task.level }}</span>
          <span class="tag tag-gray">{{ task.module }}</span>
          <span class="tag tag-orange">{{ task.ability }}</span>
        </div>
        <div class="th-tags">
          <span class="kp" v-for="k in knowledgePoints" :key="k">{{ k }}</span>
          <template v-if="linkedEthics.length">
            <span class="ethic-sep">职业伦理</span>
            <span class="ethic-trigger" v-for="s in linkedEthics" :key="s.topic" @click="goEthics">{{ s.topic }}</span>
          </template>
        </div>
      </div>
      <div class="th-r">
        <div class="th-meta">{{ headMeta }}</div>
        <div class="stepper" v-if="task.type === 'analysis'">
          <div
            v-for="(s, i) in STEPS" :key="s"
            class="step" :class="{ active: i === curStep, done: i < curStep }"
            @click="gotoPanel(GOTO[i])"
          >
            <span class="dot">{{ i < curStep ? '✓' : i + 1 }}</span>{{ s }}
          </div>
        </div>
      </div>
    </div>

    <!-- 工作区 -->
    <div class="ws">

      <!-- ===== 分析型任务：七步面板 ===== -->
      <div class="main-col" v-if="task.type === 'analysis'">
        <div class="panel" id="panel-data">
          <h3>① 经营数据（智诚科技 2026 上半年，单位：万元）</h3>
          <div class="table-wrap">
            <table>
              <thead><tr><th>月份</th><th>营业收入</th><th>营业成本</th><th>毛利率</th><th>销售费用</th><th>管理费用</th><th>净利润</th></tr></thead>
              <tbody>
                <tr v-for="d in monthData" :key="d.m">
                  <td>{{ d.m }}</td><td>{{ d.rev }}</td><td>{{ d.cost }}</td>
                  <td>{{ gm(d.rev, d.cost) }}%</td><td>{{ d.sell }}</td><td>{{ d.mng }}</td><td>{{ d.profit }}</td>
                </tr>
              </tbody>
            </table>
          </div>
          <div class="kpi-row tri mt8">
            <div class="kpi"><div class="k">6月毛利率</div><div class="v warn">{{ gmArr[5] }}%</div></div>
            <div class="kpi"><div class="k">6月销售费用率</div><div class="v warn">{{ fr(monthData[5].sell, monthData[5].rev) }}%</div></div>
            <div class="kpi"><div class="k">净利润变动</div><div class="v down">{{ profitChg }}%</div></div>
            <div class="kpi"><div class="k">收入变动</div><div class="v up">+{{ monthData[5].rev - monthData[0].rev }}万</div></div>
            <div class="kpi"><div class="k">毛利率变动</div><div class="v down">{{ gm(monthData[0].rev, monthData[0].cost) }}% → {{ gmArr[5] }}%</div></div>
            <div class="kpi"><div class="k">费用率变动</div><div class="v warn">{{ fr(monthData[0].sell, monthData[0].rev) }}% → {{ fr(monthData[5].sell, monthData[5].rev) }}%</div></div>
          </div>
        </div>

        <div class="panel">
          <h3>② 智能异常标注 <span class="panel-sub">AI 已自动标记 {{ anomalies.length }} 处关键信号</span></h3>
          <div class="anomaly" v-for="(a, i) in anomalies" :key="i" :class="{ warn: a.sev === 'mid' }">
            <div class="ai">!</div>
            <div><div class="at">{{ a.t }}</div><div class="ad">{{ a.d }}</div></div>
          </div>
        </div>

        <div class="panel" id="panel-trend">
          <h3>③ 趋势分析</h3>
          <svg class="svg-chart" viewBox="0 0 520 160" role="img">
            <text :x="28" y="14" font-size="11" fill="#5A6577" font-weight="600">净利润（万元）</text>
            <line v-for="g in gridYs" :key="g" :x1="28" :y1="g" :x2="492" :y2="g" stroke="#EEF2F8" />
            <polygon :points="chartAreaPoints(profitArr)" fill="#2B6CD622" />
            <polyline :points="chartPointsStr(profitArr)" fill="none" stroke="#2B6CD6" stroke-width="2" stroke-linejoin="round" />
            <circle v-for="(p, i) in chartPts(profitArr)" :key="i" :cx="p[0]" :cy="p[1]" r="3" fill="#2B6CD6" />
            <text v-for="(l, i) in labels" :key="'l' + i" :x="chartX(i)" y="152" font-size="10" fill="#97A1B2" text-anchor="middle">{{ l }}</text>
          </svg>
          <svg class="svg-chart mt8" viewBox="0 0 520 160" role="img">
            <text :x="28" y="14" font-size="11" fill="#5A6577" font-weight="600">毛利率（%）</text>
            <line v-for="g in gridYs" :key="'g' + g" :x1="28" :y1="g" :x2="492" :y2="g" stroke="#EEF2F8" />
            <polygon :points="chartAreaPoints(gmArr)" fill="#FA8C1622" />
            <polyline :points="chartPointsStr(gmArr)" fill="none" stroke="#FA8C16" stroke-width="2" stroke-linejoin="round" />
            <circle v-for="(p, i) in chartPts(gmArr)" :key="'p' + i" :cx="p[0]" :cy="p[1]" r="3" fill="#FA8C16" />
            <text v-for="(l, i) in labels" :key="'x' + i" :x="chartX(i)" y="152" font-size="10" fill="#97A1B2" text-anchor="middle">{{ l }}</text>
          </svg>
          <div class="fee-block">
            <div class="fee-title">销售费用率走势（6 月达峰值）</div>
            <div class="fee-row" v-for="f in feeRates" :key="f.m">
              <span class="fee-m">{{ f.m }}</span>
              <div class="bar-bg"><div class="bar-fill f-orange" :style="{ width: f.pct + '%' }"></div></div>
              <span class="fee-v">{{ f.v }}%</span>
            </div>
          </div>
          <div class="trend-insight">
            <div class="ti-head">
              <span class="ti-t">AI 趋势解读</span>
              <span class="ti-sub">· 基于 6 个月数据自动生成</span>
              <button class="btn btn-ghost btn-sm" @click="aiPlaceholder">刷新解读</button>
            </div>
            <ul class="ti-list">
              <li v-for="(t, i) in insights" :key="i">{{ t }}</li>
            </ul>
          </div>
          <div class="ethic-inline" v-if="linkedEthics.length">
            <span class="ethic-note">本任务关联职业伦理情境：</span>
            <span class="ethic-trigger" v-for="s in linkedEthics" :key="'e' + s.topic" @click="goEthics">{{ s.topic }}</span>
            <span class="ethic-note">· 点击查看情境与引导思考</span>
          </div>
          <div class="my-findings">
            <div class="mf-title">我的发现 <span class="ti-sub">· 勾选你确认的关键信号，将带入结论</span></div>
            <div class="find-chips">
              <button
                v-for="(a, i) in anomalies" :key="'f' + i"
                class="fchip" :class="{ on: draft.findings.includes(i) }"
                @click="toggleFinding(i)"
              >
                <span class="ck">{{ draft.findings.includes(i) ? '✓' : '' }}</span>{{ a.t }}
              </button>
            </div>
          </div>
        </div>

        <div class="panel">
          <div class="sg-head">
            <span class="sg-title">④ 分析思路引导</span>
            <span class="ti-sub">完成度 {{ guidePct }}%</span>
          </div>
          <div
            class="chk-row" :class="{ on: guideChecked.includes(i) }"
            v-for="(s, i) in ASTEPS" :key="i" @click="toggleGuide(i)"
          >
            <span class="ck">{{ guideChecked.includes(i) ? '✓' : '' }}</span><span class="t">{{ s }}</span>
          </div>
          <div class="bar-bg tall mt8"><div class="bar-fill f-blue" :style="{ width: guidePct + '%' }"></div></div>
        </div>

        <div class="panel">
          <h3>⑤ 快速测算工具</h3>
          <div class="calc">
            <div class="calc-field">
              <label>选择月份</label>
              <select v-model.number="calcMonth">
                <option v-for="(d, i) in monthData" :key="'m' + i" :value="i">{{ d.m }}</option>
              </select>
            </div>
            <span class="calc-tip">基于上方数据实时计算关键指标</span>
          </div>
          <div class="calc-out">
            <div class="co"><div class="co-k">毛利率</div><div class="co-v">{{ calcGm }}%</div></div>
            <div class="co"><div class="co-k">销售费用率</div><div class="co-v">{{ calcFee }}%</div></div>
          </div>
        </div>

        <div class="panel" id="panel-conc">
          <h3>⑥ 结论撰写</h3>
          <div class="ref-chips">
            参考趋势发现：
            <span class="kp" v-if="!draft.findings.length">（可在 ③ 趋势分析勾选你的发现）</span>
            <span class="kp" v-for="i in draft.findings" :key="'r' + i">{{ anomalies[i].t }}</span>
          </div>
          <div class="field">
            <label>① 核心结论（利润下降的主因是什么）</label>
            <textarea v-model="draft.core" placeholder="例：利润下降主因为销售费用率由 8.1% 升至 12.2%，叠加毛利率走低…"></textarea>
          </div>
          <div class="field">
            <label>② 数据支撑（列出关键证据）</label>
            <textarea v-model="draft.evidence" placeholder="例：净利润由 110 万降至 75 万；6 月销售费用率 12.2% 为上半年峰值…"></textarea>
          </div>
          <div class="field">
            <label>③ 改进建议（给出可执行动作）</label>
            <textarea v-model="draft.adv" placeholder="例：优化渠道投放结构、压降低效促销、建立销售费用率预警…"></textarea>
          </div>
          <div class="conc-foot">
            <div class="bar-bg tall" style="flex:1;max-width:240px"><div class="bar-fill f-blue" :style="{ width: concPct + '%' }"></div></div>
            <span class="conc-pct">完整度 {{ concPct }}%</span>
            <button class="btn btn-ghost btn-sm" @click="aiOutline">AI 生成提纲</button>
          </div>
        </div>

        <div class="panel" id="panel-submit">
          <h3>⑦ 提交评分</h3>
          <div class="submit-card">
            <div class="sc-row"><span class="muted">待提交内容</span><span>趋势发现 {{ draft.findings.length }} · 结论 {{ concLen }} 字</span></div>
            <div class="sc-preview" :class="{ empty: !draft.core }">
              {{ draft.core || '尚未填写核心结论…' }}
            </div>
            <div class="sc-actions">
              <button class="btn btn-primary" @click="submitAnalysis">提交并评分</button>
              <button class="btn btn-ghost" @click="go('student-home')">返回任务地图</button>
            </div>
          </div>
        </div>
      </div>

      <!-- ===== 测验型任务：选择题 ===== -->
      <div class="main-col" v-else-if="task.type === 'quiz'">
        <div class="panel">
          <h3>随堂测验 · 选择题 <span class="panel-sub">点击选项即时查看解析，巩固现金流 / 偿债 / 营运能力知识点</span></h3>
        </div>
        <div class="card quiz-card" v-for="(q, qi) in QUIZ_BANK" :key="q.id">
          <div class="section-title"><span class="bar"></span>第 {{ q.id }} 题 <span class="tag tag-blue">财务大数据分析</span></div>
          <div class="quiz-q">{{ q.q }}</div>
          <div class="quiz-opts">
            <div
              class="quiz-opt"
              v-for="(o, oi) in q.opts" :key="oi"
              :class="optClass(qi, oi)"
              @click="pickQuiz(qi, oi)"
            >
              <span class="qk">{{ o.split('、')[0] }}</span>
              <span>{{ o.split('、').slice(1).join('、') }}</span>
            </div>
          </div>
          <div class="quiz-fb" v-if="quizSel[qi] !== undefined">
            <div class="fb-h">
              <span :class="quizSel[qi] === q.ans ? 'fb-ok' : 'fb-no'">
                {{ quizSel[qi] === q.ans ? '✓ 回答正确' : '✗ 回答错误，正确答案 ' + q.opts[q.ans].split('、')[0] }}
              </span>
            </div>
            <div class="fb-ex">{{ q.explain }}</div>
          </div>
        </div>
        <div class="submit-row">
          <button class="btn btn-primary" @click="submitQuiz">提交并评分</button>
          <button class="btn btn-ghost" @click="go('student-home')">返回任务地图</button>
          <span class="submit-tip">提交后进入评学反馈（得分 / 能力雷达 / 错因分析）</span>
        </div>
      </div>

      <!-- ===== 计算型任务：综合财报计算 ===== -->
      <div class="main-col" v-else-if="task.type === 'calc'">
        <div class="panel">
          <h3>① 年度财报数据（某公司 2024–2025，单位：万元）</h3>
          <div class="table-wrap">
            <table>
              <thead><tr><th>项目</th><th>2024年</th><th>2025年</th></tr></thead>
              <tbody>
                <tr v-for="r in CALC_TASK.table.slice(1)" :key="r[0]">
                  <td>{{ r[0] }}</td><td>{{ r[1] }}</td><td>{{ r[2] }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
        <div class="panel">
          <div class="section-title"><span class="bar"></span>{{ CALC_TASK.title }}</div>
          <div class="req-title">计算要求</div>
          <ul class="req-list">
            <li v-for="(r, i) in CALC_TASK.requirements" :key="i">{{ r }}</li>
          </ul>
        </div>
        <div class="grid-2 calc-cols">
          <div class="card">
            <div class="section-title"><span class="bar"></span>我的计算结果 <span class="panel-sub">点击"核对答案"比对</span></div>
            <div class="calc-grid">
              <div class="field" v-for="f in calcFields" :key="f.id">
                <label>{{ f.label }}</label>
                <input v-model="myCalc[f.id]" :placeholder="f.ph" />
              </div>
            </div>
            <button class="btn btn-primary btn-sm mt8" @click="checkCalc">核对答案</button>
            <div class="quiz-fb" v-if="calcChecked">
              <div class="fb-h"><span class="fb-ok">共 6 项指标，正确 {{ calcCorrect }} 项</span></div>
              <div class="table-wrap mt8">
                <table>
                  <thead><tr><th>指标</th><th>我的结果</th><th>标准值</th><th>判定</th></tr></thead>
                  <tbody>
                    <tr v-for="r in calcRows" :key="r.label">
                      <td>{{ r.label }}</td><td>{{ r.got || '—' }}</td><td>{{ r.target }}</td>
                      <td :class="r.good ? 'fb-ok' : 'fb-no'">{{ r.good ? '✓ 正确' : '✗ 偏差' }}</td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </div>
          </div>
          <div class="card" v-if="calcChecked">
            <div class="section-title"><span class="bar"></span>标准答案参考</div>
            <div class="calc-ans">
              <div class="ca-row" v-for="r in calcRows" :key="'a' + r.label"><span>{{ r.label }}</span><b>{{ r.target }}</b></div>
            </div>
            <div class="trend-insight mt8">
              <div class="ti-head"><span class="ti-t">AI 分析提示</span></div>
              <ul class="ti-list">
                <li>偿债能力：流动比率 {{ ans.currentRatio }}、速动比率 {{ ans.quickRatio }}，短期偿债安全边际尚可；资产负债率升至 {{ ans.debtRatio }}%，长期偿债压力加大。</li>
                <li>营运能力：应收账款周转率 {{ ans.art }} 次（约 {{ ans.artDays }} 天回款），整体周转健康。</li>
                <li>盈利能力：营业净利率 {{ ans.netProfitMargin }}%、总资产收益率 {{ ans.roa }}%，盈利水平稳健。</li>
                <li>利润质量：经营现金净流量由 1200 万降至 800 万，净利润增长但现金流下滑，利润"含金量"下降，需关注赊销与营运资本占用。</li>
              </ul>
            </div>
          </div>
        </div>
        <div class="submit-row">
          <button class="btn btn-primary" @click="submitCalc">提交并评分</button>
          <button class="btn btn-ghost" @click="go('student-home')">返回任务地图</button>
          <span class="submit-tip">提交后进入评学反馈（得分 / 能力雷达 / 错因分析）</span>
        </div>
      </div>

      <!-- ===== AI 导师列（占位对话） ===== -->
      <div class="ai-col">
        <div class="ai-panel">
          <div class="mentor-head">
            <span class="mentor-avatar">AI</span>
            <div class="mentor-title">
              <b>AI 财务导师</b>
              <small>炼技 Agent · 伴学答疑（演示占位）</small>
            </div>
            <button class="btn-ghost btn-sm" @click="resetMentor">清空</button>
          </div>
          <div class="mentor-chips">
            <button class="qchip" v-for="q in mentorQuestions" :key="q" @click="sendMentor(q)">{{ q }}</button>
          </div>
          <div class="mentor-hist">
            <div v-for="(m, i) in chatMsgs" :key="i" class="chat-msg" :class="{ user: m.role === 'user' }">
              <span class="chat-av">{{ m.role === 'user' ? '我' : 'AI' }}</span>
              <div class="chat-bubble">{{ m.text }}</div>
            </div>
          </div>
          <div class="mentor-input">
            <input v-model="chatInput" placeholder="向导师提问，如：毛利率下降有哪些原因？" @keydown.enter="sendMentor()" />
            <button class="btn btn-primary btn-sm" @click="sendMentor()">发送</button>
          </div>
          <div class="ai-extra" v-if="task.type === 'analysis'">
            <button class="btn btn-ghost btn-sm" @click="getHint">获取分层提示</button>
          </div>
          <div class="hint-stack">
            <div class="hint-box" v-for="(h, i) in hintsShown" :key="i">
              <div class="hi">分层提示 {{ i + 1 }}/{{ HINTS.length }}</div>{{ h }}
            </div>
          </div>
          <div class="mentor-foot">AI 导师功能待接入：请在 AI_KEY 填入智能体 API Key</div>
        </div>
      </div>
    </div>

    <!-- 提示 -->
    <div v-if="tip" class="tip">{{ tip }}</div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from "vue";

const emit = defineEmits(["navigate"]);

// ============ 数据服务（localStorage 共享，PRESET 兜底） ============
function lsGet(k) { try { const v = localStorage.getItem(k); return v ? JSON.parse(v) : null; } catch (e) { return null; } }
function lsSet(k, v) { try { localStorage.setItem(k, JSON.stringify(v)); } catch (e) { /* 忽略 */ } }
const LS_STU = "fd_student_v1";

// ============ AI 智能体配置（占位，接入后替换 AI_KEY） ============
const AI_URL = "https://agent.gjt-smart.com/v1/chat-messages";
const AI_KEY = "app-xxxxxx"; // TODO 占位

const PRESET_STUDENT = {
  name: "林晓", cls: "财管2201", level: 3, lvName: "数据分析员", xp: 640, xpMax: 1000,
  tasks: [
    { id: "T01", name: "财报数据治理：多系统对账与清洗", level: "初级", module: "M1", ability: "数据治理", status: "done", score: 92 },
    { id: "T02", name: "三大报表结构解读与勾稽关系", level: "初级", module: "M1", ability: "报表理解", status: "done", score: 88 },
    { id: "T03", name: "毛利率异常分析与费用率排查", level: "初级", module: "M2", ability: "指标分析", type: "analysis", status: "active", score: null },
    { id: "T04", name: "多维度经营解析（产品/地区/渠道）", level: "中级", module: "M2", ability: "多维分析", type: "analysis", status: "locked" },
    { id: "T05", name: "异常特征识别与帕累托分析", level: "中级", module: "M3", ability: "营运能力", type: "analysis", status: "locked" },
    { id: "T06", name: "成本结构量价拆分模型", level: "中级", module: "M3", ability: "成本控制", type: "analysis", status: "locked" },
    { id: "T07", name: "现金流分析与利润质量评价", level: "中级", module: "M4", ability: "现金流分析", type: "analysis", status: "locked" },
    { id: "T08", name: "偿债能力分层与风险预警诊断", level: "高级", module: "M4", ability: "风险特警", type: "analysis", status: "locked" },
    { id: "T09", name: "非经常性损益与宏观环境影响分析", level: "高级", module: "M5", ability: "综合分析", type: "analysis", status: "locked" },
    { id: "T10", name: "动态仪表盘搭建与BI看板优化", level: "高级", module: "M6", ability: "可视化", type: "analysis", status: "locked" },
    { id: "T11", name: "随堂测验·选择题（现金流/偿债/营运）", level: "初级", module: "M2", ability: "指标分析", type: "quiz", status: "done", score: 100 },
    { id: "T12", name: "随堂练习·综合财报计算题", level: "中级", module: "M4", ability: "综合分析", type: "calc", status: "done", score: 92 }
  ],
  currentTask: null,
  lastFeedback: null,
  badges: [
    { n: "数据侦探", d: "发现关键异常指标", earned: true },
    { n: "报表达人", d: "准确解读三大报表", earned: true },
    { n: "成本管家", d: "完成成本优化任务", earned: false },
    { n: "图表设计师", d: "高质量可视化分析", earned: true },
    { n: "经营参谋", d: "提出有效改进建议", earned: false },
    { n: "财务洞察官", d: "完成综合经营挑战", earned: false }
  ],
  leaderboard: [
    { nm: "王浩", xp: 880, cls: "财管2201" }, { nm: "林晓", xp: 640, cls: "财管2201" },
    { nm: "陈雨", xp: 610, cls: "财管2201" }, { nm: "李娜", xp: 555, cls: "财管2202" },
    { nm: "赵磊", xp: 498, cls: "财管2201" }, { nm: "周婷", xp: 460, cls: "财管2202" }
  ],
  streak: 5, accuracy: 89, lastActive: "今天 09:24",
  recommend: {
    id: "T04", name: "多维度经营解析（产品/地区/渠道）", ability: "多维分析",
    reason: "你已完成毛利率与费用率分析（M2 前半），下一步建议做多维度经营拆解，补齐「多维分析」短板后进入 M3 异常特征识别。"
  },
  abilityProg: [
    { k: "报表理解", v: 92 }, { k: "指标分析", v: 85 }, { k: "异常识别", v: 70 },
    { k: "现金流分析", v: 64 }, { k: "可视化表达", v: 88 }, { k: "风险预警", v: 58 }
  ],
  abilityRadar: {
    axes: ["财报数据治理", "经营解析能力", "异常特征识别", "现金流分析", "风险预警能力", "可视化建模"],
    cur: [78, 82, 76, 70, 58, 84], avg: [75, 72, 74, 70, 66, 71]
  },
  ethicsScenes: [
    { topic: "数据真实性", kp: "数据真实性核验", theme: "数据诚信", title: "数字背后的责任",
      scene: "你在录入某上市公司三季度营收数据时发现：把一笔 1,200 万的收入错计到下季度，本月营收指标恰好「达标」。系统提示「数据异常波动」，你是否修正？",
      guide: ["若小数点或归属期错配，可能误导怎样的经营决策？", "数据真实为什么是财务人的第一底线？", "遇到「达标」诱惑时，应如何守住职业底线？"],
      case: "《会计法》明确要求会计核算以实际发生的经济业务为依据——虚构、提前确认收入均属违法。" },
    { topic: "财务造假", kp: "利润质量评价", theme: "行业伦理", title: "瑞幸式陷阱",
      scene: "你正在评价某公司利润质量，发现其净利润连年增长但经营现金流持续为负、应收账款激增。导师提示：这可能是「纸面富贵」。",
      guide: ["利润与现金流背离意味着什么？", "收入确认激进可能踩中哪条监管红线？", "如何从财报勾稽关系识别财务造假迹象？"],
      case: "瑞幸咖啡虚构 22 亿元交易额被重罚——财务造假的代价远超短期股价收益。" },
    { topic: "盈余管理", kp: "费用率分析", theme: "职业判断", title: "费用的「挪移」",
      scene: "季度末，部门为完成利润目标建议你将一笔市场费用顺延至下季度确认。报表会「更好看」，但这是否触碰合规边界？",
      guide: ["费用跨期确认与财务造假的法律边界在哪？", "盈余管理与企业长期价值的冲突？", "如果你是财务总监，会怎么做？"],
      case: "某企业财务主管因通过费用跨期进行盈余管理被处罚——真实代价远比想象更大。" },
    { topic: "数据隐私", kp: "财报数据治理", theme: "数据安全", title: "报表里的敏感信息",
      scene: "你导出的分析表中包含客户名单、供应商结算价等未脱敏字段，准备分享给外部咨询团队。是否存在泄露风险？",
      guide: ["哪些财报字段属于敏感/受限数据？", "对外提供数据前应做哪些脱敏？", "泄露商业秘密的法律后果是什么？"],
      case: "《数据安全法》《个人信息保护法》对商业秘密与个人信息处理有明确要求，违规将追责。" },
    { topic: "算法偏差", kp: "可视化建模", theme: "科技伦理", title: "模型不会说谎，但人会",
      scene: "你搭建的信用评分模型在测试中发现：对某区域客户系统性给出更低评分。这可能源于训练数据的历史偏见。",
      guide: ["算法偏差从何而来？", "分析师对模型结果负有什么责任？", "如何在建模中识别并修正偏差？"],
      case: "模型偏差若不加审视，会把历史不公固化为「客观结论」——分析师须做偏差审查。" },
    { topic: "商业秘密", kp: "财报数据治理", theme: "职业操守", title: "未公开的预告",
      scene: "你提前看到尚未披露的毛利率下滑数据，朋友恰好持有该公司股票并询问「最近业绩怎么样」。你该如何回应？",
      guide: ["内幕信息知情人的保密义务是什么？", "随意透露未公开财报可能构成什么行为？", "如何建立「知密不言」的职业习惯？"],
      case: "利用未公开财务信息交易属内幕交易，知情人负有法定保密与回避义务。" }
  ],
  feed: [
    { icon: "check", txt: "完成「三大报表结构解读与勾稽关系」，得分 88", time: "2 天前" },
    { icon: "star", txt: "解锁「图表设计师」徽章", time: "3 天前" },
    { icon: "trophy", txt: "班级排名上升至第 2 名", time: "本周" },
    { icon: "bot", txt: "AI 导师为你推荐「多维度经营解析」", time: "今天" }
  ]
};

function loadStudent() {
  let s = lsGet(LS_STU);
  if (!s) { s = JSON.parse(JSON.stringify(PRESET_STUDENT)); lsSet(LS_STU, s); }
  return s;
}

function loadTask(s) {
  if (!s) return null;
  let t = s.currentTask || null;
  if (!t) t = (s.tasks || []).find(function (x) { return x.status === "active"; }) || null;
  if (!t) t = (s.tasks || [])[2] || null;
  return t;
}

// ============ 静态数据：月度经营数据（M2 · 常规经营解析） ============
const monthData = [
  { m: '1月', rev: 520, cost: 310, sell: 42, mng: 58, profit: 110 },
  { m: '2月', rev: 498, cost: 305, sell: 45, mng: 60, profit: 88 },
  { m: '3月', rev: 535, cost: 322, sell: 48, mng: 62, profit: 103 },
  { m: '4月', rev: 560, cost: 340, sell: 55, mng: 66, profit: 99 },
  { m: '5月', rev: 572, cost: 352, sell: 63, mng: 70, profit: 87 },
  { m: '6月', rev: 581, cost: 360, sell: 71, mng: 75, profit: 75 }
];

// ============ 选择题库（随堂测验） ============
const QUIZ_BANK = [
  { id: 14, q: '某公司经营活动现金净流量大于零，但利润为负值，这可能说明（　　）。',
    opts: ['A、企业盈利能力很强', 'B、企业可能收回了大量以前年度的应收账款', 'C、企业大量赊销导致收入虚高', 'D、企业已陷入财务危机'], ans: 1,
    explain: '经营活动现金净流量>0说明企业在经营中实打实收到了现金；利润<0可能是会计口径的亏损（如大额折旧、减值准备等非现金支出）。收回往期应收账款会带来正向现金流但不影响当期利润。' },
  { id: 15, q: '某公司年末流动资产为1400万元，年末流动比率为2，年末速动比率为1，则年末存货余额为（　　）。',
    opts: ['A、30万元', 'B、70万元', 'C、90万元', 'D、120万元'], ans: 1,
    explain: '流动比率=流动资产/流动负债=2 → 流动负债=700万；速动比率=1 → 速动资产=700万；速动资产=流动资产-存货 → 存货=1400-700=700万。教材经典口径下选B。' },
  { id: 16, q: '某公司年末资产总额为6000万元，产权比率为5，则资产负债率为（　　）。',
    opts: ['A、65.33%', 'B、70.25%', 'C、83.33%', 'D、85.25%'], ans: 2,
    explain: '产权比率=负债/所有者权益=5 → 负债=5×权益；资产=负债+权益=6×权益=6000 → 权益=1000万，负债=5000万；资产负债率=5000/6000=83.33%。' },
  { id: 17, q: '某企业流通在外的普通股股数为2亿股，每股交易价格为13.2元，印花税税率为1%，每股收益为0.4元，则市盈率为（　　）。',
    opts: ['A、3.3', 'B、33', 'C、0.33', 'D、23'], ans: 1,
    explain: '市盈率=每股市价/每股收益=13.2/0.4=33倍。印花税税率是干扰信息。' }
];

// ============ 综合计算题（M4 · 综合财报计算） ============
const CALC_TASK = {
  title: '某公司2025年度财务报表综合分析',
  table: [
    ['项目', '2024年', '2025年'],
    ['营业收入（万元）', '8000', '10000'],
    ['营业成本（万元）', '4800', '6500'],
    ['净利润（万元）', '1000', '1200'],
    ['流动资产（万元）', '4000', '5000'],
    ['流动负债（万元）', '2500', '3200'],
    ['存货（万元）', '1000', '1500'],
    ['资产总额（万元）', '10000', '12000'],
    ['负债总额（万元）', '5000', '6800'],
    ['所有者权益总额（万元）', '5000', '5200'],
    ['应收账款（万元）', '600', '800'],
    ['经营活动现金净流量（万元）', '1200', '800']
  ],
  requirements: [
    '计算该公司2025年的以下财务指标：流动比率、速动比率、资产负债率、营业净利率、总资产收益率（平均资产总额）、应收账款周转率（平均应收账款，一年按360天计算）。',
    '结合上述指标，对公司的偿债能力、营运能力、盈利能力进行分析。',
    '结合经营活动现金净流量的变化，评价公司的利润质量上升还是下降。'
  ],
  answer: {
    currentRatio: (5000 / 3200).toFixed(2),
    quickRatio: ((5000 - 1500) / 3200).toFixed(2),
    debtRatio: (6800 / 12000 * 100).toFixed(2),
    netProfitMargin: (1200 / 10000 * 100).toFixed(2),
    roa: (1200 / ((10000 + 12000) / 2) * 100).toFixed(2),
    art: (10000 / ((600 + 800) / 2)).toFixed(2),
    artDays: (360 / 14.29).toFixed(0)
  }
};

// ============ 任务知识点映射（课程六大模块） ============
const KNOW = {
  '报表理解': ['三大报表结构', '科目勾稽关系', '数据治理清洗'],
  '数据治理': ['多系统对账', '异常值处理', 'Power BI/Excel ETL'],
  '指标分析': ['毛利率/费用率', '环比同比趋势', '阈值预警'],
  '费用分析': ['销售费用率', '费用弹性系数', '量价拆分法'],
  '多维分析': ['产品/地区/渠道维度', '结构拆解', '帕累托分析'],
  '营运能力': ['应收账款周转率', '存货周转率', '账龄分析'],
  '成本控制': ['量价拆分模型', '单位成本变动', '变动成本法'],
  '现金流分析': ['OCF净流量', '自由现金流', '利润质量'],
  '风险特警': ['Z-score模型', '资金链预警', '偿债能力分层'],
  '综合分析': ['杜邦分析体系', '雷达图诊断', '根因定位树'],
  '报告表达': ['分析报告结构', '图表规范', '结论表述'],
  '决策支持': ['情景模拟', '敏感性分析', '管理建议闭环'],
  '岗位认知': ['财务分析师职责', '大数据工具链', '职业道德'],
  '可视化': ['动态仪表盘', 'BI看板搭建', '模型迭代优化']
};

// ============ 任务 → 职业伦理情境锚定（按课程模块映射） ============
const ETHICS_LINK = {
  M1: ['数据真实性', '数据隐私', '商业秘密'],
  M2: ['盈余管理'],
  M4: ['财务造假'],
  M6: ['算法偏差']
};

// ============ 分层提示（炼技 Agent 四级学习支架演示） ============
const HINTS = [
  '先查看经营看板，找出 6 个月中环比变化最大的指标。',
  '把"收入、成本、费用"三个维度分别拆开对比，定位利润下滑的主因。',
  '计算各月毛利率与费用率，验证"销售费用率走高"是否为主要原因，再写结论。'
];

const CALC_FIELDS = [
  { id: 'cr', label: '流动比率', ph: '例 1.56' },
  { id: 'qr', label: '速动比率', ph: '例 1.09' },
  { id: 'dr', label: '资产负债率(%)', ph: '例 56.67' },
  { id: 'nm', label: '营业净利率(%)', ph: '例 12.00' },
  { id: 'roa', label: '总资产收益率(%)', ph: '例 10.91' },
  { id: 'art', label: '应收账款周转率(次)', ph: '例 14.29' }
];
const calcFields = CALC_FIELDS;

// ============ 工具函数 ============
function gm(r, c) { return ((r - c) / r * 100).toFixed(1); }
function fr(s, r) { return (s / r * 100).toFixed(1); }

// ============ 页面状态 ============
const student = ref(loadStudent());
const task = ref(loadTask(student.value));
if (task.value && !student.value.currentTask) {
  student.value.currentTask = task.value;
  lsSet(LS_STU, student.value);
}
const draft = ref(task.value.draft || (task.value.draft = { findings: [], core: '', evidence: '', adv: '' }));
const tip = ref("");
let tipTimer = null;
function showTip(t) { tip.value = t; clearTimeout(tipTimer); tipTimer = setTimeout(function () { tip.value = ""; }, 2400); }
function persist() { lsSet(LS_STU, student.value); }
function refresh() {
  student.value = loadStudent();
  task.value = loadTask(student.value);
  if (task.value && !student.value.currentTask) { student.value.currentTask = task.value; }
  persist();
  showTip("数据已刷新");
}

// ============ 任务面板状态 ============
const STEPS = ['数据研读', '趋势分析', '结论撰写', '提交评分'];
const GOTO = ['panel-data', 'panel-trend', 'panel-conc', 'panel-submit'];
const ASTEPS = [
  '看趋势：对比 6 个月指标变化，找出波动最大项',
  '找异常：定位偏离正常阈值的指标',
  '拆维度：按收入 / 成本 / 费用分别拆解',
  '查原因：关联业务事件解释变化',
  '提建议：给出可执行经营改进动作'
];
const curStep = ref(1);
const guideChecked = ref([]);
const calcMonth = ref(monthData.length - 1);
const hintsShown = ref([]);
const quizSel = ref({});
const myCalc = ref({ cr: '', qr: '', dr: '', nm: '', roa: '', art: '' });
const calcChecked = ref(false);

// ============ 计算属性 ============
const knowledgePoints = computed(function () { return KNOW[task.value.ability] || ['关键指标', '分析方法', '业务含义']; });
const linkedEthics = computed(function () {
  const topics = ETHICS_LINK[task.value.module] || [];
  return student.value.ethicsScenes.filter(function (s) { return topics.indexOf(s.topic) >= 0; });
});
const headMeta = computed(function () {
  if (task.value.type === 'quiz') return '共 ' + QUIZ_BANK.length + ' 道选择题 · 覆盖现金流 / 偿债 / 营运能力 · 点击选项查看解析';
  if (task.value.type === 'calc') return '综合财报计算题 · 完成测算后点击「核对答案」比对标准结果';
  return '预计时长 约 45 分钟 · 截止 本周日 23:59 · 目标：找出利润下降的关键影响因素';
});
const labels = computed(function () { return monthData.map(function (d) { return d.m; }); });
const profitArr = computed(function () { return monthData.map(function (d) { return d.profit; }); });
const gmArr = computed(function () { return monthData.map(function (d) { return +gm(d.rev, d.cost); }); });
const profitChg = computed(function () { return (((monthData[5].profit - monthData[0].profit) / monthData[0].profit) * 100).toFixed(1); });
const feeRates = computed(function () {
  const max = Math.max.apply(null, monthData.map(function (d) { return +fr(d.sell, d.rev); }));
  return monthData.map(function (d) {
    const v = +fr(d.sell, d.rev);
    return { m: d.m, v: v, pct: Math.round(v / max * 100) };
  });
});
const anomalies = computed(function () {
  const d = monthData;
  const pChg = (((d[5].profit - d[0].profit) / d[0].profit) * 100).toFixed(1);
  const rChg = (((d[5].rev - d[0].rev) / d[0].rev) * 100).toFixed(1);
  return [
    { sev: 'high', t: '净利润持续下滑', d: '上半年由 ' + d[0].profit + ' 万降至 ' + d[5].profit + ' 万（' + pChg + '%），6 月环比仍降，需定位主因。' },
    { sev: 'high', t: '销售费用率攀升', d: '由 ' + fr(d[0].sell, d[0].rev) + '% 升至 ' + fr(d[5].sell, d[5].rev) + '%，6 月达上半年峰值，直接吞噬利润。' },
    { sev: 'mid', t: '增收不增利', d: '收入增长 ' + rChg + '%，利润反降 ' + pChg + '%；量价背离提示成本/费用失控。' },
    { sev: 'mid', t: '毛利率走低', d: '由 ' + gm(d[0].rev, d[0].cost) + '% 降至 ' + gm(d[5].rev, d[5].cost) + '%，建议拆"收入"与"成本"两维度定位原因。' }
  ];
});
const insights = computed(function () {
  const d = monthData;
  const pChg = Math.round((d[0].profit - d[5].profit) / d[0].profit * 100);
  const rChg = ((d[5].rev - d[0].rev) / d[0].rev * 100).toFixed(1);
  return [
    '净利润由 ' + d[0].profit + ' 万降至 ' + d[5].profit + ' 万，上半年累计下滑约 ' + pChg + '%。',
    '销售费用率由 ' + fr(d[0].sell, d[0].rev) + '% 升至 ' + fr(d[5].sell, d[5].rev) + '%，6 月为上半年峰值。',
    '毛利率由 ' + gm(d[0].rev, d[0].cost) + '% 降至 ' + gm(d[5].rev, d[5].cost) + '%，成本端持续承压。',
    '收入增长 ' + rChg + '% 但利润反降，呈典型"增收不增利"。'
  ];
});
const guidePct = computed(function () { return Math.round(guideChecked.value.length / ASTEPS.length * 100); });
const calcGm = computed(function () { const d = monthData[calcMonth.value]; return gm(d.rev, d.cost); });
const calcFee = computed(function () { const d = monthData[calcMonth.value]; return fr(d.sell, d.rev); });
const concLen = computed(function () { return (draft.value.core + draft.value.evidence + draft.value.adv).length; });
const concPct = computed(function () {
  const n = [draft.value.core, draft.value.evidence, draft.value.adv].filter(function (x) { return x.trim(); }).length;
  return Math.round(n / 3 * 100);
});
const ans = computed(function () { return CALC_TASK.answer; });
const calcRows = computed(function () {
  const a = ans.value;
  const map = [
    ['cr', a.currentRatio, '流动比率'], ['qr', a.quickRatio, '速动比率'], ['dr', a.debtRatio, '资产负债率'],
    ['nm', a.netProfitMargin, '营业净利率'], ['roa', a.roa, '总资产收益率'], ['art', a.art, '应收账款周转率']
  ];
  return map.map(function (m) {
    const v = (myCalc.value[m[0]] || '').trim();
    const got = parseFloat(v);
    return { label: m[2], target: m[1], got: v, good: v !== '' && !isNaN(got) && Math.abs(got - parseFloat(m[1])) < 0.05 };
  });
});
const calcCorrect = computed(function () { return calcRows.value.filter(function (r) { return r.good; }).length; });

// ============ 内联 SVG 折线图（自适应宽度） ============
const CHART_W = 520, CHART_H = 160, CHART_PAD = 28;
function chartX(i) { return CHART_PAD + (CHART_W - CHART_PAD * 2) * i / Math.max(labels.value.length - 1, 1); }
function chartBounds(vals) {
  return { min: Math.min.apply(null, vals) * 0.9, max: Math.max.apply(null, vals) * 1.1 };
}
function chartY(v, min, max) { return CHART_H - CHART_PAD - (CHART_H - CHART_PAD * 2) * (v - min) / (max - min || 1); }
function chartPts(vals) {
  const b = chartBounds(vals);
  return vals.map(function (v, i) { return [chartX(i), chartY(v, b.min, b.max)]; });
}
function chartPointsStr(vals) {
  return chartPts(vals).map(function (p) { return p.join(','); }).join(' ');
}
function chartAreaPoints(vals) {
  const pts = chartPts(vals);
  if (!pts.length) return '';
  const b = chartBounds(vals);
  const base = chartY(b.min, b.min, b.max);
  return CHART_PAD + ',' + base + ' ' + chartPointsStr(vals) + ' ' + (CHART_W - CHART_PAD) + ',' + base;
}
const gridYs = computed(function () { return [0, 0.5, 1].map(function (g) { return CHART_PAD + (CHART_H - CHART_PAD * 2) * g; }); });

// ============ 交互 ============
function go(view) { emit("navigate", view); }
function goEthics() { go('student-ethics'); }
function gotoPanel(id) {
  const el = document.getElementById(id);
  if (el) el.scrollIntoView({ behavior: 'smooth', block: 'start' });
}
function toggleFinding(i) {
  const idx = draft.value.findings.indexOf(i);
  if (idx >= 0) draft.value.findings.splice(idx, 1);
  else draft.value.findings.push(i);
}
function toggleGuide(i) {
  const idx = guideChecked.value.indexOf(i);
  if (idx >= 0) guideChecked.value.splice(idx, 1);
  else guideChecked.value.push(i);
}
function aiPlaceholder() { showTip("AI 功能待接入：请在 AI_KEY 填入智能体 API Key"); }
function getHint() {
  if (hintsShown.value.length >= HINTS.length) { showTip("已无更多提示"); return; }
  hintsShown.value.push(HINTS[hintsShown.value.length]);
  aiPlaceholder();
}
function aiOutline() { aiPlaceholder(); }
function unlockNext() {
  const nx = student.value.tasks.find(function (x) { return x.status === 'locked'; });
  if (nx) nx.status = 'active';
}

// ---------- 分析型任务 ----------
function submitAnalysis() {
  const d = draft.value;
  if (!d.core.trim() || !d.evidence.trim() || !d.adv.trim()) { showTip("请完整填写核心结论 / 数据支撑 / 改进建议"); return; }
  task.value.status = 'done';
  task.value.score = 89;
  task.value.conclusion = { core: d.core, evidence: d.evidence, adv: d.adv, findings: [].concat(d.findings) };
  student.value.lastFeedback = {
    taskId: task.value.id, type: 'analysis',
    sub: '本次「' + task.value.name + '」已完成自动评分与 AI 复盘',
    dims: [
      { label: '任务完成度', full: 20, got: 20 }, { label: '数据处理准确性', full: 20, got: 18 },
      { label: '指标分析能力', full: 20, got: 17 }, { label: '业务逻辑合理性', full: 15, got: 12 },
      { label: '图表表达效果', full: 10, got: 9 }, { label: '报告规范性', full: 10, got: 8 },
      { label: '创新与决策建议', full: 5, got: 5 }
    ],
    radar: null,
    conclusion: task.value.conclusion,
    errTags: ['结论缺少业务逻辑', '图表选择不当'],
    errNote: 'AI 诊断：你已正确识别销售费用率上升是主因，但对"产品结构变化"分析不足，建议补充不同产品毛利率对比。',
    aiComment: '优点：指标计算准确、趋势判断到位。建议：补做产品维度毛利率拆解；下个推荐任务 → 产品盈利能力分析（T05）。',
    xpGain: 60
  };
  unlockNext();
  persist();
  showTip("提交成功，AI 正在评分…");
  setTimeout(function () { go('student-feedback'); }, 400);
}

// ---------- 测验型任务 ----------
function pickQuiz(qi, oi) { quizSel.value[qi] = oi; }
function optClass(qi, oi) {
  if (quizSel.value[qi] === undefined) return '';
  if (oi === QUIZ_BANK[qi].ans) return 'correct';
  if (oi === quizSel.value[qi]) return 'wrong';
  return '';
}
function submitQuiz() {
  const total = QUIZ_BANK.length;
  for (let i = 0; i < total; i++) {
    if (quizSel.value[i] === undefined) { showTip("请完成第 " + (i + 1) + " 题后再提交"); return; }
  }
  const correctArr = QUIZ_BANK.map(function (q, i) { return quizSel.value[i] === q.ans; });
  const correct = correctArr.filter(Boolean).length;
  const score = Math.round(correct / total * 100);
  const topics = ['现金流与利润质量', '短期偿债能力', '长期偿债能力', '估值与指标辨析'];
  student.value.lastFeedback = {
    taskId: task.value.id, type: 'quiz',
    sub: '本次「' + task.value.name + '」已完成自动评分与 AI 复盘',
    dims: topics.map(function (tp, i) { return { label: tp, full: 25, got: correctArr[i] ? 25 : 0 }; }),
    radar: { axes: topics, cur: topics.map(function (_, i) { return correctArr[i] ? 100 : 55; }), avg: [80, 75, 82, 78] },
    overview: '共 ' + total + ' 题，答对 ' + correct + ' 题，综合得分 ' + score + '。',
    overviewRows: topics.map(function (tp, i) {
      return { label: tp, ok: correctArr[i], answer: QUIZ_BANK[i].opts[QUIZ_BANK[i].ans].split('、')[0] };
    }),
    errTags: topics.filter(function (tp, i) { return !correctArr[i]; }).map(function (tp) { return tp + ' 待巩固'; }),
    errNote: correct < total
      ? 'AI 诊断：选择题侧重概念辨析与公式应用，建议结合课程能力图谱回顾错项对应知识点（如现金流性质、偿债比率勾稽）。'
      : 'AI 诊断：选择题全部正确，概念辨析扎实，建议进入综合计算题深化指标测算能力。',
    aiComment: '优点：选择题作答准确率 ' + score + '%，基础概念掌握' + (correct === total ? '扎实' : '良好') + '。建议：进入综合财报计算题，强化指标测算与综合分析。',
    xpGain: 40
  };
  finishTask(score);
}

// ---------- 计算型任务 ----------
function checkCalc() {
  const missing = calcRows.value.filter(function (r) { return !r.got; }).length;
  if (missing > 0) { showTip("请填写全部 6 项指标后再核对（还差 " + missing + " 项）"); return; }
  calcChecked.value = true;
}
function submitCalc() {
  if (!calcChecked.value) { showTip("请先点击「核对答案」完成比对"); return; }
  const axes = ['流动比率', '速动比率', '资产负债率', '营业净利率', '总资产收益率', '应收账款周转率'];
  const correct = calcCorrect.value;
  const score = Math.round(correct / 6 * 100);
  const fulls = [17, 17, 17, 17, 16, 16];
  student.value.lastFeedback = {
    taskId: task.value.id, type: 'calc',
    sub: '本次「' + task.value.name + '」已完成自动评分与 AI 复盘',
    dims: axes.map(function (ax, i) { return { label: ax, full: fulls[i], got: calcRows.value[i].good ? fulls[i] : 0 }; }),
    radar: { axes: axes, cur: axes.map(function (_, i) { return calcRows.value[i].good ? 100 : 55; }), avg: [80, 78, 75, 82, 79, 76] },
    overview: '6 项指标，正确 ' + correct + ' 项，综合得分 ' + score + '。',
    calcRows: calcRows.value,
    errTags: axes.filter(function (ax, i) { return !calcRows.value[i].good; }).map(function (ax) { return ax + ' 计算偏差'; }),
    errNote: correct < 6
      ? 'AI 诊断：指标计算存在偏差，建议复核公式（流动比率=流动资产/流动负债；速动比率=(流动资产-存货)/流动负债；资产负债率=负债/资产），并注意"平均资产 / 平均应收"口径。'
      : 'AI 诊断：六项指标全部计算正确，测算能力扎实。建议结合现金流变化评价利润质量。',
    aiComment: '优点：综合财报计算准确率 ' + score + '%，指标测算' + (correct === 6 ? '精准' : '基本到位') + '。建议：将计算结果用于偿债能力、营运能力、盈利能力联动分析，并评价利润质量。',
    xpGain: 40
  };
  finishTask(score);
}
function finishTask(score) {
  task.value.status = 'done';
  task.value.score = score;
  unlockNext();
  persist();
  showTip("提交成功，AI 正在评分…");
  setTimeout(function () { go('student-feedback'); }, 400);
}

// ---------- AI 导师（占位对话） ----------
const mentorWelcome = '你好，我是你的 AI 财务分析导师。本任务目标是找出利润下降的关键影响因素，你可以问我指标含义、分析方法，或请求分层提示。';
const mentorQuestions = ['这些指标分别怎么算？', '流动比率怎么看？', '利润质量怎么评价？', '怎么提经营建议？'];
const chatMsgs = ref([{ role: 'ai', text: mentorWelcome }]);
const chatInput = ref('');
function sendMentor(q) {
  const t = (q || chatInput.value || '').trim();
  if (!t) return;
  chatInput.value = '';
  chatMsgs.value.push({ role: 'user', text: t });
  aiPlaceholder();
  setTimeout(function () {
    chatMsgs.value.push({ role: 'ai', text: '（占位回复）AI 导师对话功能待接入。接入后可在 AI_KEY 填入智能体 API Key，基于本任务数据流式解答。' });
  }, 500);
}
function resetMentor() { chatMsgs.value = [{ role: 'ai', text: mentorWelcome }]; }

// ---------- 样式辅助 ----------
function lvTag(lv) { return lv === "初级" ? "tag-blue" : lv === "中级" ? "tag-orange" : "tag-red"; }

onMounted(function () { });
</script>

<style scoped>
/* ============ 基线（对齐学习主页 / Ant 风格） ============ */
.pg { max-width: 1100px; margin: 0 auto; padding: 18px 16px 40px; }
.page-head { display: flex; align-items: center; gap: 10px; margin-bottom: 16px; flex-wrap: wrap; }
.page-head-bar { width: 4px; height: 22px; border-radius: 2px; background: #1677FF; }
.page-head-title { font-size: 20px; font-weight: 600; color: #1F2733; }
.page-head-sub { font-size: 13px; color: #97A1B2; flex: 1; }
.btn { display: inline-flex; align-items: center; gap: 6px; height: 34px; padding: 0 16px; border-radius: 6px; font-size: 14px; font-weight: 600; border: 1px solid transparent; cursor: pointer; transition: all .15s ease; white-space: nowrap; }
.btn:disabled { opacity: .5; cursor: not-allowed; }
.btn-primary { background: #2B6CD6; color: #fff; }
.btn-primary:hover { background: #4D8BE8; }
.btn-ghost { color: #2B6CD6; background: #EAF2FC; }
.btn-ghost:hover { background: #DCEBFB; }
.btn-light { background: #fff; color: #2B6CD6; }
.btn-sm { height: 28px; padding: 0 12px; font-size: 13px; }
.mt { margin-top: 16px; }
.mt2 { margin-top: 20px; }
.mt8 { margin-top: 8px; }
.muted { color: #5A6577; font-size: 13px; }
.card { background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 16px; box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05); }
.section-title { display: flex; align-items: center; gap: 8px; font-size: 15px; font-weight: 600; color: #1F2733; margin-bottom: 14px; }
.section-title .bar { width: 4px; height: 15px; border-radius: 2px; background: #2B6CD6; }
.grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
.kpi-row { display: grid; grid-template-columns: repeat(5, 1fr); gap: 12px; }
.kpi-row.tri { grid-template-columns: repeat(3, 1fr); }
.kpi { background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 13px 15px; }
.kpi .k { font-size: 12px; color: #5A6577; }
.kpi .v { font-size: 21px; font-weight: 600; margin-top: 2px; color: #1F2733; }
.kpi .v small { font-size: 12px; color: #97A1B2; font-weight: 400; }
.v.up { color: #52C41A; }
.v.down { color: #FF4D4F; }
.v.warn { color: #FAAD14; }
.tag { display: inline-flex; align-items: center; font-size: 12px; padding: 1px 8px; border-radius: 4px; font-weight: 500; white-space: nowrap; }
.tag-blue { background: #EAF2FC; color: #1D4FA8; }
.tag-orange { background: #FFF3E8; color: #C96A06; }
.tag-red { background: #FFECEC; color: #D93025; }
.tag-green { background: #F1FAE8; color: #3C8E12; }
.tag-gray { background: #F7FAFD; color: #97A1B2; }
.tag-light { background: rgba(255, 255, 255, .25); color: #fff; }
.bar-bg { background: #EEF3FA; border-radius: 99px; overflow: hidden; }
.bar-bg.tall { height: 8px; }
.bar-fill { height: 100%; border-radius: 99px; transition: width .4s ease; }
.f-blue { background: #2B6CD6; }
.f-orange { background: #FA8C16; }
.f-red { background: #FF4D4F; }
.tip { margin-top: 12px; background: #EAF2FC; border: 1px solid #DCEBFB; color: #1D4FA8; font-size: 13px; border-radius: 6px; padding: 8px 12px; }
.empty { padding: 40px; text-align: center; color: #97A1B2; font-size: 13px; }
@media (max-width: 900px) { .grid-2 { grid-template-columns: 1fr; } .kpi-row, .kpi-row.tri { grid-template-columns: repeat(2, 1fr); } .ws { grid-template-columns: 1fr; } .ai-col { position: static; } .calc-cols { grid-template-columns: 1fr; } }

/* ============ 任务头部 ============ */
.task-head { display: flex; flex-wrap: wrap; gap: 14px; align-items: flex-start; justify-content: space-between; margin-bottom: 16px; }
.th-l { flex: 1; min-width: 240px; }
.th-title { display: flex; align-items: center; gap: 8px; flex-wrap: wrap; margin-bottom: 10px; }
.th-name { font-size: 18px; font-weight: 600; color: #1F2733; }
.th-tags { display: flex; flex-wrap: wrap; gap: 6px; align-items: center; }
.th-r { flex: 1; min-width: 300px; }
.th-meta { font-size: 12px; color: #5A6577; margin-bottom: 10px; text-align: right; }
.kp { font-size: 12px; background: #fff; border: 1px solid #D6E3F5; color: #1D4FA8; padding: 2px 9px; border-radius: 99px; }
.ethic-sep { font-size: 12px; color: #97A1B2; font-weight: 600; margin-left: 6px; }
.ethic-trigger {
  display: inline-flex; align-items: center; gap: 5px; padding: 3px 10px;
  border: 1px solid #2B6CD6; background: #EAF2FC; color: #1D4FA8;
  border-radius: 99px; font-size: 12px; font-weight: 600; cursor: pointer; transition: .15s; white-space: nowrap;
}
.ethic-trigger:hover { background: #2B6CD6; color: #fff; }
.ethic-note { font-weight: 400; }

/* 步骤条 */
.stepper { display: flex; align-items: center; flex-wrap: wrap; }
.step {
  display: flex; align-items: center; gap: 7px; font-size: 12px; color: #97A1B2;
  position: relative; padding-right: 18px; cursor: pointer;
}
.step .dot {
  width: 22px; height: 22px; border-radius: 50%; background: #EEF3FA; color: #97A1B2;
  display: flex; align-items: center; justify-content: center; font-weight: 600; font-size: 12px; flex-shrink: 0;
}
.step:not(:last-child)::after {
  content: ""; position: absolute; left: 30px; right: 0; top: 11px; height: 2px; background: #E3E9F2;
}
.step.active { color: #1D4FA8; font-weight: 600; }
.step.active .dot { background: #2B6CD6; color: #fff; }
.step.done .dot { background: #52C41A; color: #fff; }
.step.done { color: #5A6577; }

/* ============ 工作区双栏 ============ */
.ws { display: grid; grid-template-columns: 1fr 340px; gap: 16px; align-items: start; }
.main-col { min-width: 0; }
.ai-col { position: sticky; top: 0; }
.ai-panel {
  background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 16px;
  box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05);
  display: flex; flex-direction: column; gap: 10px;
}

/* ============ 面板 ============ */
.panel {
  background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 18px;
  box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05); margin-bottom: 16px; scroll-margin-top: 12px;
}
.panel h3 { font-size: 15px; font-weight: 600; margin-bottom: 12px; }
.panel-sub { font-size: 12px; font-weight: 400; color: #97A1B2; }

/* 表格 */
.table-wrap { overflow-x: auto; border: 1px solid #E3E9F2; border-radius: 8px; }
.table-wrap table { width: 100%; border-collapse: collapse; font-size: 13px; font-variant-numeric: tabular-nums; }
.table-wrap th, .table-wrap td { padding: 9px 12px; border-bottom: 1px solid #EEF2F8; text-align: right; }
.table-wrap th:first-child, .table-wrap td:first-child { text-align: left; }
.table-wrap thead th { background: #F7FAFD; color: #1F2733; font-weight: 600; white-space: nowrap; }
.table-wrap tbody tr:last-child td { border-bottom: none; }
.table-wrap tbody tr:hover td { background: #F7FAFD; }

/* 折线图 */
.svg-chart { width: 100%; display: block; }

/* 异常标注 */
.anomaly {
  display: flex; gap: 11px; padding: 10px 12px; border-radius: 6px;
  background: #FFF7F5; border: 1px solid #FBD9CE; margin-bottom: 9px;
}
.anomaly:last-child { margin-bottom: 0; }
.anomaly .ai {
  width: 26px; height: 26px; border-radius: 7px; flex-shrink: 0;
  background: #FFECEC; color: #FF4D4F;
  display: flex; align-items: center; justify-content: center; margin-top: 1px; font-weight: 700; font-size: 14px;
}
.anomaly .at { font-weight: 600; font-size: 13px; color: #B23A1E; }
.anomaly .ad { font-size: 12px; color: #5A6577; margin-top: 2px; line-height: 1.5; }
.anomaly.warn { border-color: #FBE3B0; background: #FEFBF2; }
.anomaly.warn .ai { background: #FFF7E0; color: #FAAD14; }
.anomaly.warn .at { color: #9A6A00; }

/* 费用率条 */
.fee-block { margin-top: 14px; }
.fee-title { font-size: 13px; font-weight: 600; margin-bottom: 10px; }
.fee-row { display: flex; align-items: center; gap: 10px; margin-bottom: 7px; }
.fee-m { width: 30px; font-size: 12px; color: #5A6577; flex-shrink: 0; }
.fee-row .bar-bg { flex: 1; height: 8px; }
.fee-v { width: 44px; text-align: right; font-size: 12px; color: #5A6577; font-variant-numeric: tabular-nums; flex-shrink: 0; }

/* 趋势洞察 */
.trend-insight { margin-top: 14px; background: #EAF2FC; border: 1px solid #D6E3F5; border-radius: 6px; padding: 12px 14px; }
.ti-head { display: flex; align-items: center; gap: 6px; font-weight: 600; font-size: 13px; color: #1D4FA8; margin-bottom: 8px; }
.ti-t { font-weight: 600; }
.ti-sub { font-weight: 400; color: #97A1B2; font-size: 12px; }
.ti-head .btn { margin-left: auto; }
.ti-list { margin: 0; padding-left: 18px; font-size: 13px; color: #5A6577; line-height: 1.8; list-style: disc; }
.ti-list li { margin-bottom: 8px; }
.ethic-inline {
  margin-top: 14px; display: flex; align-items: center; gap: 6px; flex-wrap: wrap;
  font-size: 12px; color: #5A6577; background: #EAF2FC;
  border: 1px solid #D6E3F5; border-radius: 6px; padding: 9px 12px;
}
.ethic-inline .ethic-trigger { border-style: dashed; background: transparent; }

/* 我的发现 */
.my-findings { margin-top: 14px; }
.mf-title { font-size: 13px; font-weight: 600; margin-bottom: 8px; }
.find-chips { display: flex; flex-wrap: wrap; gap: 8px; }
.fchip {
  display: inline-flex; align-items: center; gap: 6px; padding: 6px 11px; border-radius: 99px;
  border: 1px solid #E3E9F2; background: #fff; font-size: 12px; cursor: pointer; transition: .15s; color: #5A6577;
}
.fchip:hover { border-color: #2B6CD6; }
.fchip.on { background: #EAF2FC; border-color: #2B6CD6; color: #1D4FA8; font-weight: 600; }
.fchip .ck {
  width: 15px; height: 15px; border-radius: 5px; border: 1.5px solid #E3E9F2;
  display: inline-flex; align-items: center; justify-content: center; flex-shrink: 0;
  color: #fff; font-size: 11px; line-height: 1;
}
.fchip.on .ck { background: #2B6CD6; border-color: #2B6CD6; }

/* 思路引导 */
.sg-head { display: flex; justify-content: space-between; align-items: center; font-size: 13px; margin-bottom: 10px; }
.sg-title { font-size: 15px; font-weight: 600; }
.chk-row {
  display: flex; align-items: center; gap: 10px; padding: 9px 10px;
  border-radius: 6px; cursor: pointer; transition: background .15s;
  border: 1px solid transparent; font-size: 13px;
}
.chk-row:hover { background: #F7FAFD; }
.chk-row .ck {
  width: 18px; height: 18px; border-radius: 5px; border: 2px solid #E3E9F2;
  flex-shrink: 0; display: inline-flex; align-items: center; justify-content: center;
  color: #fff; font-size: 12px; line-height: 1;
}
.chk-row.on { background: #EAF2FC; }
.chk-row.on .ck { background: #2B6CD6; border-color: #2B6CD6; }
.chk-row.on .t { color: #1D4FA8; font-weight: 500; }

/* 快速测算 */
.calc { display: flex; align-items: center; gap: 12px; flex-wrap: wrap; }
.calc-field { display: flex; align-items: center; gap: 8px; }
.calc-field label { font-size: 13px; font-weight: 600; }
.calc-field select { height: 34px; border: 1px solid #E3E9F2; border-radius: 6px; padding: 0 11px; background: #fff; color: #1F2733; }
.calc-field select:focus { outline: none; border-color: #2B6CD6; box-shadow: 0 0 0 2px #EAF2FC; }
.calc-tip { font-size: 12px; color: #97A1B2; }
.calc-out { display: flex; gap: 18px; margin-top: 12px; flex-wrap: wrap; }
.calc-out .co { background: #F7FAFD; border: 1px solid #EEF2F8; border-radius: 6px; padding: 10px 14px; min-width: 120px; }
.calc-out .co .co-k { font-size: 12px; color: #5A6577; }
.calc-out .co .co-v { font-size: 20px; font-weight: 700; color: #2B6CD6; margin-top: 2px; }

/* 结论撰写 */
.ref-chips {
  display: flex; flex-wrap: wrap; align-items: center; gap: 6px;
  background: #F7FAFD; border-radius: 6px;
  padding: 9px 11px; margin-bottom: 12px; font-size: 12px;
}
.field { margin-bottom: 14px; }
.field label { display: block; font-size: 13px; font-weight: 600; margin-bottom: 6px; }
.field input, .field select, .field textarea {
  width: 100%; height: 34px; border: 1px solid #E3E9F2; border-radius: 6px; padding: 0 11px;
  background: #fff; color: #1F2733; transition: border-color .15s, box-shadow .15s;
}
.field textarea { height: auto; padding: 8px 11px; resize: vertical; min-height: 70px; }
.field input:focus, .field select:focus, .field textarea:focus { outline: none; border-color: #2B6CD6; box-shadow: 0 0 0 2px #EAF2FC; }
.conc-foot { display: flex; align-items: center; gap: 12px; margin-top: 6px; flex-wrap: wrap; }
.conc-pct { font-size: 12px; color: #5A6577; white-space: nowrap; }

/* 提交卡 */
.submit-card { background: #F7FAFD; border: 1px solid #EEF2F8; border-radius: 6px; padding: 14px; }
.sc-row { display: flex; justify-content: space-between; font-size: 12px; margin-bottom: 8px; flex-wrap: wrap; gap: 6px; }
.sc-preview {
  font-size: 13px; line-height: 1.6; color: #1F2733;
  background: #fff; border-radius: 6px; padding: 10px 12px; min-height: 42px;
}
.sc-preview.empty { color: #97A1B2; }
.sc-actions { display: flex; gap: 10px; margin-top: 12px; flex-wrap: wrap; }
.submit-row { display: flex; gap: 10px; align-items: center; margin-top: 4px; flex-wrap: wrap; }
.submit-tip { font-size: 12px; color: #97A1B2; }

/* ============ 选择题 ============ */
.quiz-card { margin-bottom: 16px; }
.quiz-q { font-size: 14px; line-height: 1.7; margin-bottom: 12px; }
.quiz-opts { display: flex; flex-direction: column; gap: 8px; }
.quiz-opt {
  display: flex; align-items: center; gap: 11px; padding: 11px 14px;
  border: 1px solid #E3E9F2; border-radius: 6px; cursor: pointer; transition: .15s; font-size: 14px;
}
.quiz-opt:hover { border-color: #2B6CD6; background: #EAF2FC; }
.quiz-opt .qk {
  width: 24px; height: 24px; border-radius: 50%; background: #EEF3FA;
  color: #5A6577; display: flex; align-items: center; justify-content: center;
  font-weight: 600; font-size: 13px; flex-shrink: 0;
}
.quiz-opt.correct { border-color: #52C41A; background: #F1FAE8; }
.quiz-opt.correct .qk { background: #52C41A; color: #fff; }
.quiz-opt.wrong { border-color: #FF4D4F; background: #FFECEC; }
.quiz-opt.wrong .qk { background: #FF4D4F; color: #fff; }
.quiz-fb {
  margin-top: 12px; background: #F7FAFD; border: 1px solid #EEF2F8;
  border-radius: 6px; padding: 12px 14px; font-size: 13px; line-height: 1.7; color: #1F2733;
}
.quiz-fb .fb-h { font-weight: 600; margin-bottom: 6px; }
.fb-ok { color: #52C41A; }
.fb-no { color: #FF4D4F; }
.quiz-fb .fb-ex { color: #5A6577; margin-top: 4px; }

/* ============ 综合计算题 ============ */
.req-title { font-size: 15px; font-weight: 600; margin: 0 0 10px; color: #1F2733; }
.req-list { padding-left: 20px; font-size: 13px; line-height: 1.8; color: #5A6577; list-style: decimal; margin: 0; }
.calc-cols { margin-bottom: 16px; }
.calc-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
.calc-grid .field { margin: 0; }
.calc-ans { display: flex; flex-direction: column; gap: 9px; }
.ca-row {
  display: flex; justify-content: space-between; align-items: center;
  padding: 8px 12px; background: #F7FAFD; border: 1px solid #EEF2F8;
  border-radius: 6px; font-size: 13px;
}
.ca-row span { color: #5A6577; }
.ca-row b { font-size: 15px; color: #2B6CD6; font-variant-numeric: tabular-nums; }

/* ============ AI 导师面板（占位对话） ============ */
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
.mentor-hist { display: flex; flex-direction: column; gap: 10px; max-height: 320px; overflow-y: auto; padding: 4px 2px; }
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
  padding: 8px 12px; border-radius: 10px; font-size: 13px; line-height: 1.7; word-break: break-word;
}
.chat-msg.user .chat-bubble { background: #2B6CD6; border-color: #2B6CD6; color: #fff; }
.mentor-input { display: flex; gap: 8px; padding-top: 12px; border-top: 1px solid #E3E9F2; margin-top: 2px; }
.mentor-input input {
  flex: 1; height: 32px; border: 1px solid #E3E9F2; border-radius: 6px; padding: 0 12px; font-size: 13px;
}
.mentor-input input:focus { outline: none; border-color: #2B6CD6; box-shadow: 0 0 0 2px #EAF2FC; }
.mentor-foot { font-size: 11px; color: #97A1B2; text-align: center; }
.ai-extra { padding-top: 10px; border-top: 1px solid #E3E9F2; }
.hint-stack { display: flex; flex-direction: column; gap: 8px; }
.hint-box {
  background: #EAF2FC; border: 1px solid #DCEBFB; border-radius: 6px;
  padding: 10px 12px; font-size: 12.5px; color: #1D4FA8; line-height: 1.6;
  animation: hintIn .25s ease;
}
@keyframes hintIn { from { opacity: 0; transform: translateY(-4px); } to { opacity: 1; transform: none; } }
.hint-box .hi { font-weight: 600; margin-bottom: 4px; }
</style>
