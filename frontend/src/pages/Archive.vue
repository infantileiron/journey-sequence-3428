<script setup lang="ts">
import {ref} from 'vue';
import * as reviews from '../api/reviews';
import * as reports from '../api/reports';
import * as runs from '../api/runs';
import {download} from '../api/client';
import {useResource} from '../hooks/useResource';
import {refreshResources} from '../stores/resources';
import {user} from '../stores/session';
import PageTitle from '../components/common/PageTitle.vue';
import Notice from '../components/common/Notice.vue';
import StatusBadge from '../components/common/StatusBadge.vue';
import type {Review,Report} from '../types/entities';

const {data:rv,error}=useResource(reviews.list,[] as Review[]);
const {data:rp}=useResource(reports.list,[] as Report[]);
const {data:rr}=useResource(runs.list,[]);
const runId=ref(sessionStorage.getItem('journey-selected-run')||'run-baseline');
const comment=ref('已核验支持度去重、分组口径与编辑成本，请复核。');
const message=ref('');
const withdrawId=ref('');
const withdrawReason=ref('');

async function submit(){
  try{await reviews.create({run_id:runId.value,comment:comment.value});refreshResources();message.value='';}
  catch(e:any){message.value=e.message;}
}
async function decide(id:string,status:string){
  try{
    await reviews.decide(id,{status,comment:status==='approved'?'支持度与编辑对齐口径已核验，通过归档，保留描述性解释边界。':'请补充旅程分母及关联解释边界说明后重新提交。'});
    refreshResources();message.value='';
  }catch(e:any){message.value=e.message;}
}
function startWithdraw(id:string){withdrawId.value=id;withdrawReason.value='';message.value='';}
function cancelWithdraw(){withdrawId.value='';withdrawReason.value='';}
async function submitWithdraw(id:string){
  if(withdrawReason.value.trim().length<5){message.value='请填写撤回原因（至少5个字），说明解释有误的具体情况。';return;}
  try{
    await reviews.withdraw(id,withdrawReason.value.trim());
    withdrawId.value='';withdrawReason.value='';refreshResources();message.value='';
  }catch(e:any){message.value=e.message;}
}
async function publish(id:string){
  try{
    const r=rv.value.find(x=>x.id===id)!;
    await reports.create({title:'栖山观展路径顺序策展图册',run_id:r.run_id,review_id:r.id});
    refreshResources();message.value='';
  }catch(e:any){message.value=e.message;}
}
async function getReport(id:string){
  try{await download('/reports/'+id+'/download','栖山观展路径顺序.csv');}
  catch(e:any){message.value=e.message;}
}
const activeReports=()=>rp.value.filter(r=>r.downloadable);
const stoppedReports=()=>rp.value.filter(r=>!r.downloadable);
</script>

<template>
  <PageTitle chapter="06" title="把路径依据，一起存档" description="分析员提交，策展复核人批准。解释有误可填写原因撤回批准：旧报告保留历史但停止下载，重新送审并再次批准后仅新报告可下载。"/>
  <Notice :error="error||message"/>

  <form v-if="user?.role!=='viewer'" class="paper form-grid" @submit.prevent="submit">
    <label>完成的运行
      <select v-model="runId" aria-label="完成的运行">
        <option v-for="r in rr.filter(x=>x.status==='completed')" :key="r.id" :value="r.id">{{r.id}}</option>
      </select>
    </label>
    <label>策展备注<textarea v-model="comment"/></label>
    <button class="primary">提交路径复核</button>
  </form>

  <section class="paper">
    <div class="section-intro">
      <h2>复核记录</h2>
      <p>待复核 / 已批准 / 已退回 / 已撤回 全程留痕，撤回后可对同一运行重新送审</p>
    </div>
    <article v-for="r in rv" :key="r.id" class="review-row">
      <StatusBadge :status="r.status"/>
      <div>
        <b>{{r.run_id}}</b>
        <p>{{r.comment}}</p>
        <small>{{r.reviewer||'等待策展复核'}}</small>
        <div v-if="r.status==='withdrawn'" class="withdraw-box">
          <b>撤回原因</b>
          <p>{{r.withdraw_reason}}</p>
          <small>撤回人：{{r.withdraw_reviewer}} · {{r.withdrawn_at}}</small>
        </div>
        <form v-if="withdrawId===r.id" class="withdraw-form" @submit.prevent="submitWithdraw(r.id)">
          <label>撤回原因（将随复核记录归档，必填）
            <textarea v-model="withdrawReason" placeholder="说明原批准解释有误的具体情况，至少5个字"></textarea>
          </label>
          <div class="actions">
            <button class="primary" type="submit">确认撤回批准</button>
            <button type="button" @click="cancelWithdraw">取消</button>
          </div>
        </form>
      </div>
      <div class="actions">
        <template v-if="user?.role==='admin'&&r.status==='pending'">
          <button @click="decide(r.id,'approved')">批准方案</button>
          <button @click="decide(r.id,'rejected')">退回复查</button>
        </template>
        <template v-if="user?.role==='admin'&&r.status==='approved'">
          <button v-if="withdrawId!==r.id" @click="startWithdraw(r.id)">撤回批准</button>
        </template>
        <button v-if="user?.role!=='viewer'&&r.status==='approved'&&withdrawId!==r.id" @click="publish(r.id)">生成归档报告</button>
        <small v-if="r.status==='withdrawn'||r.status==='rejected'" class="resubmit-hint">可在上方表单对该运行重新提交复核</small>
      </div>
    </article>
  </section>

  <section class="paper">
    <div class="section-intro">
      <h2>有效图册（可下载）</h2>
      <p>仅引用当前仍有效批准复核的报告可下载；旧报告不会因重新批准而恢复</p>
    </div>
    <p v-if="!activeReports().length" class="empty">当前没有可下载的有效报告</p>
    <article v-for="r in activeReports()" :key="r.id" class="record report-row">
      <div>
        <b>{{r.title}}</b>
        <small>{{r.run_id}} · 复核 {{r.review_id}}</small>
      </div>
      <button @click="getReport(r.id)">下载顺序 CSV ↓</button>
    </article>
  </section>

  <section class="paper">
    <div class="section-intro">
      <h2>已停止图册（历史保留，不可下载）</h2>
      <p>复核撤回后旧报告记录仍可查阅，但已停止下载且不再生成新文件</p>
    </div>
    <p v-if="!stoppedReports().length" class="empty">暂无被撤回或失效的历史报告</p>
    <article v-for="r in stoppedReports()" :key="r.id" class="record report-row stopped">
      <div>
        <b>{{r.title}}</b>
        <small>{{r.run_id}} · 复核 {{r.review_id}}</small>
        <StatusBadge :status="r.review_status||'withdrawn'"/>
      </div>
      <button disabled title="引用的复核已撤回，报告停止下载">下载已停止</button>
    </article>
  </section>
</template>
