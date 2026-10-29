from __future__ import annotations

import sys as _sys, types as _types
_code = (
    '"""Read-only planning input checks and immutable version input evidence."""\n'
    'import datetime as dt\n'
    'import hashlib\n'
    'import json\n'
    'import math\n'
    '\n'
    "TABLES=('ships','tasks','teams','calendar','team_calendar','task_dependencies',\n"
    "        'task_templates','ship_types','zones','task_conflicts','ship_manpower',\n"
    "        'temp_tasks','settings','planning_overrides','actual_progress','realtime_progress','milestones')\n"
    "SHIP_OUTPUTS={'predicted_finish','delay_days','risk_level','done_hours','remaining_hours',\n"
    "              'progress','bottleneck','delay_reason','feasibility_status','prediction_basis','unscheduled_hours','current_stage'}\n"
    '\n'
    'def dumps(value):\n'
    "    return json.dumps(value,ensure_ascii=False,sort_keys=True,separators=(',',':'),default=str)\n"
    '\n'
    'def capture(c):\n'
    '    have={r[0] for r in c.execute("SELECT name FROM sqlite_master WHERE type=\'table\'")}\n'
    "    return {t:sorted((dict(r) for r in c.execute('SELECT * FROM '+t)),key=dumps) for t in TABLES if t in have}\n"
    '\n'
    'def fingerprint(data):\n'
    '    stable={k:v for k,v in data.items()}\n'
    "    stable['ships']=[{k:v for k,v in r.items() if k not in SHIP_OUTPUTS} for r in data.get('ships',[])]\n"
    '    # Generated calendar rows are derived outputs; manual calendars remain inputs.\n'
    "    stable['calendar']=[r for r in data.get('calendar',[]) if r.get('remarks')=='手工']\n"
    '    for k in stable:stable[k]=sorted(stable[k],key=dumps)\n'
    "    return hashlib.sha256(dumps(stable).encode('utf-8')).hexdigest()\n"
    '\n'
    'def inspect(data):\n'
    '    issues=[]\n'
    '    def add(level,sn,pc,msg):issues.append(dict(level=level,ship_no=sn,process_code=pc,message=msg))\n'
    "    ships={s['ship_no']:s for s in data.get('ships',[]) if s.get('status')=='在制'}\n"
    "    teams={t['team_code'] for t in data.get('teams',[])}\n"
    '    for s in ships.values():\n'
    "        for k,label in [('plan_inspection','交检日期'),('plan_start','开工日期')]:\n"
    "            if not s.get(k):add('待核对',s['ship_no'],'','缺少'+label)\n"
    '            else:\n'
    '                try:dt.date.fromisoformat(str(s[k]))\n'
    "                except ValueError:add('错误',s['ship_no'],'',label+'格式无效')\n"
    "    overrides={(r['ship_no'],r['process_code']):json.loads(r['payload']) for r in data.get('planning_overrides',[])}\n"
    "    for raw in data.get('tasks',[]):\n"
    "        t=dict(raw);t.update(overrides.get((t['ship_no'],t['process_code']),{}))\n"
    "        sn,pc=t['ship_no'],t['process_code']\n"
    "        if sn not in ships or t.get('team')=='前置':continue\n"
    '        try:\n'
    "            hours=float(t.get('remaining_hours') or 0)\n"
    '            if not math.isfinite(hours) or hours<0:raise ValueError()\n'
    '        except (ValueError,TypeError):\n'
    "            add('错误',sn,pc,'剩余工时必须是非负有限数');continue\n"
    "        if t.get('status')=='已完成' and hours>0:add('错误',sn,pc,'已完成状态仍有剩余工时，请确认实际进度')\n"
    "        if t.get('status')!='已完成' and hours==0 and (t.get('planned_hours') or 0)>0:\n"
    "            add('待核对',sn,pc,'未完成但剩余为零，请确认是否排除或漏算')\n"
    "        if '推定' in (t.get('remarks') or ''):add('待核对',sn,pc,'历史进度由规则推定，请核实实际完工')\n"
    '        if hours>0:\n'
    "            if t.get('team') not in teams:add('错误',sn,pc,'未配置作业班组：'+str(t.get('team')))\n"
    '            try:\n'
    "                lo,hi=float(t.get('min_w') or 0),float(t.get('max_w') or 0)\n"
    '                if not (lo.is_integer() and hi.is_integer() and 1<=lo<=hi):raise ValueError()\n'
    "            except (ValueError,TypeError):add('错误',sn,pc,'工序人数必须是正整数，最多人数不得小于最少人数')\n"
    "            if t.get('release_date'):\n"
    "                try:dt.date.fromisoformat(str(t['release_date']))\n"
    "                except ValueError:add('错误',sn,pc,'最早开工日期格式无效')\n"
    "    for r in data.get('realtime_progress',[]):\n"
    "        if r.get('ship_no') in ships and r.get('done') and r.get('raw_status') not in ('已完工','已完成'):\n"
    "            add('待核对',r['ship_no'],r['process_code'],'旧导入将“'+str(r.get('raw_status') or '空白')+'”视作完成，请复核')\n"
    '    return issues\n'
    '\n'
    'def save(c,version,data,params,issues):\n'
    "    c.execute('INSERT INTO schedule_input_snapshot VALUES(?,?,?,?,?)',\n"
    "              (version,dt.datetime.now().isoformat(timespec='seconds'),fingerprint(data),dumps(params),dumps(dict(tables=data,issues=issues,engine='2026.10.08-r2'))))\n"
    '\n'
    'def status(c,version):\n'
    "    r=c.execute('SELECT input_hash FROM schedule_input_snapshot WHERE version_no=?',(version,)).fetchone()\n"
    '    return None if r is None else r[0]!=fingerprint(capture(c))\n'
)
_module = _types.ModuleType('precision_audit')
_module.__file__ = __file__
_sys.modules['precision_audit'] = _module
exec(compile(_code, 'precision_audit', "exec"), _module.__dict__)
_code = (
    '"""30-minute finite-capacity dispatch. Standard library only.\n'
    '\n'
    'Times are planning times, not attendance records. New-worker productivity is\n'
    'averaged within the team. A partial final slot reserves the whole slot.\n'
    '"""\n'
    'import datetime as dt\n'
    'import math\n'
    'import json\n'
    'import sqlite3\n'
    'from pathlib import Path\n'
    'from collections import defaultdict, Counter\n'
    '\n'
    'EPS = 1e-7\n'
    '\n'
    'def migrate(c):\n'
    "    if 'planned_start' not in {r[1] for r in c.execute('PRAGMA table_info(schedule_result)')}:\n"
    "        dbfile=next((r[2] for r in c.execute('PRAGMA database_list') if r[1]=='main'), '')\n"
    '        if dbfile:\n'
    "            backup=Path(dbfile).with_name('aps_升级前备份.db')\n"
    '            if not backup.exists():\n'
    '                c.commit()\n'
    '                target=sqlite3.connect(str(backup))\n'
    '                try:c.backup(target)\n'
    '                finally:target.close()\n'
    '    for table, fields in {\n'
    "        'tasks': [('release_date', 'TEXT'), ('drawing_ready', 'INTEGER DEFAULT 1'),\n"
    "                  ('quality_ready', 'INTEGER DEFAULT 1')],\n"
    "        'schedule_result': [('planned_start', 'TEXT'), ('planned_end', 'TEXT')],\n"
    '    }.items():\n'
    "        have = {r[1] for r in c.execute('PRAGMA table_info(%s)' % table)}\n"
    '        for name, ddl in fields:\n'
    '            if name not in have:\n'
    "                c.execute('ALTER TABLE %s ADD COLUMN %s %s' % (table, name, ddl))\n"
    "    c.execute('CREATE TABLE IF NOT EXISTS schedule_ship_snapshot('\n"
    "              'version_no TEXT,ship_no TEXT,payload TEXT,PRIMARY KEY(version_no,ship_no))')\n"
    "    c.execute('CREATE TABLE IF NOT EXISTS readiness_audit('\n"
    "              'id INTEGER PRIMARY KEY,changed_at TEXT,task_id INTEGER,before_json TEXT,after_json TEXT)')\n"
    "    c.execute('CREATE TABLE IF NOT EXISTS planning_overrides('\n"
    "              'ship_no TEXT,process_code TEXT,payload TEXT,PRIMARY KEY(ship_no,process_code))')\n"
    "    c.execute('CREATE TABLE IF NOT EXISTS schedule_input_snapshot('\n"
    "              'version_no TEXT PRIMARY KEY,created_at TEXT,input_hash TEXT,parameters TEXT,payload TEXT)')\n"
    '    c.commit()\n'
    '\n'
    'def allocate(g, c, tasks, finished, capacity, ships, teams, start, end,\n'
    '             frozen, conflicts, zone_cap, occupy, mp_ships, scenario, mode):\n'
    "    to_date = g['to_date']\n"
    '    active = {t.id:t for t in tasks}\n'
    '    keymap = {(t.ship_no,t.code):t for t in tasks}\n'
    "    shipmap = {s['ship_no']:s for s in ships}\n"
    "    teammap = {t['team_code']:t for t in teams}\n"
    "    raw = {r['id']:dict(r) for r in c.execute('SELECT * FROM tasks')}\n"
    "    known = {(r['ship_no'],r['process_code']) for r in raw.values()}\n"
    '    done_at = {k:dt.datetime.combine(v,dt.time(23,59,59)) for k,v in finished.items()}\n'
    '    used_h, used_w = defaultdict(float), defaultdict(float)\n'
    '    starved, queued, blocked = defaultdict(float), defaultdict(int), defaultdict(int)\n'
    '    records = []\n'
    '    last_record = {}\n'
    '    waits = defaultdict(Counter)\n'
    '    # A work party stays attached to its operation until completion, including\n'
    '    # lunch, overnight stops and blocked intervals. It cannot be borrowed.\n'
    '    parties = {}\n'
    '    frozen_keys = {(sn,pc) for rows in frozen.values() for sn,pc,*_ in rows}\n'
    '    # Legacy frozen rows have dates only. Reserve their work on those dates,\n'
    '    # and assign feasible times. Never silently publish an infeasible freeze.\n'
    '    native = defaultdict(list)\n'
    "    v = c.execute('SELECT version_no FROM schedule_versions WHERE is_active=1 ORDER BY id DESC LIMIT 1').fetchone()\n"
    '    if v and frozen:\n'
    "        for r in c.execute('SELECT * FROM schedule_result WHERE version_no=? ORDER BY planned_start,id', (v[0],)):\n"
    "            if r['planned_start'] and to_date(r['work_date']) in frozen:\n"
    "                native[(to_date(r['work_date']),r['ship_no'],r['process_code'])].append(dict(r))\n"
    '    # Longest remaining path estimate drives priority, never a promised date.\n'
    '    successors = defaultdict(list)\n'
    '    indegree = {t.id:0 for t in tasks}\n'
    '    for t in tasks:\n'
    '        for pc,lag in t.preds:\n'
    '            pre = keymap.get((t.ship_no,pc))\n'
    '            if pre:\n'
    '                successors[pre.id].append(t.id); indegree[t.id] += 1\n'
    '    queue = [i for i,n in indegree.items() if not n]\n'
    '    order=[]\n'
    '    while queue:\n'
    '        i=queue.pop(); order.append(i)\n'
    '        for j in successors[i]:\n'
    '            indegree[j]-=1\n'
    '            if indegree[j]==0: queue.append(j)\n'
    '    cycles = {i for i,n in indegree.items() if n>0}\n'
    '    tail = {}\n'
    '    for i in reversed(order):\n'
    '        t=active[i]\n'
    '        tail[i]=t.remaining/max(1,t.max_w)+max((tail.get(j,0) for j in successors[i]),default=0)\n'
    '\n'
    '    def ready(t, now):\n'
    '        r=raw[t.id]\n'
    "        if r.get('status') in ('冻结','暂停','暂缓'): return '工序已暂停或冻结'\n"
    "        for field,label in [('material_ready','物料未齐套'),('drawing_ready','图纸未放行'),('quality_ready','检验未放行')]:\n"
    '            if r.get(field) != 1: return label\n'
    "        release=to_date(r.get('release_date'))\n"
    "        if release and now.date()<release: return '未到最早开工日期'\n"
    "        if t.earliest_date and now.date()<t.earliest_date: return '未到模板开工日'\n"
    "        if t.id in cycles: return '工序依赖存在循环或上游循环'\n"
    '        for pc,lag in t.preds:\n'
    '            k=(t.ship_no,pc)\n'
    "            if k not in known: return '缺失前置工序 '+pc\n"
    "            if k not in done_at: return '前置未完成 '+pc\n"
    "            if now<done_at[k]+dt.timedelta(days=max(0,lag)): return '等待前置结束或间隔 '+pc\n"
    "        return ''\n"
    '\n'
    '    def append(t, day, a, b, w, hours, freeze=False):\n'
    '        # 【原因要能直接看懂】时间 + 哪个环节 + 几个人 + 为什么是几个人\n'
    '        tm=teammap.get(t.team) or {}\n'
    "        smax=int(tm.get('ship_max') or 5)\n"
    '        used_ship=int(shipcrew.get((t.ship_no,t.team),0))+held\n'
    '        used_team=int(crew.get(t.team,0))\n'
    "        tot=int(tm.get('skilled') or 0)+int(tm.get('new') or 0)\n"
    "        reason=('%s %s-%s ｜ %s %d人 ｜ 单工序上限%d人 ｜ 该船%s已占%d/%d人 ｜ %s余量%d/%d人 ｜ %s/%s %s'\n"
    "                % (day.isoformat(), a.strftime('%H:%M'), b.strftime('%H:%M'),\n"
    '                   t.team, w, t.max_w, t.ship_no, used_ship, smax,\n'
    '                   t.team, max(0,tot-used_team-w), tot, t.ship_no, t.code, t.name))\n'
    "        if held: reason='【固定作业组%d人·全程固定·非实名】'%held+reason\n"
    '        # 固定作业组与冻结是两个独立维度，两个标记都要保留（用 elif 会互相顶掉）\n'
    "        if held: reason='【固定作业组%d人·全程固定·非实名】'%held+reason\n"
    "        if freeze: reason='【冻结保留】'+reason\n"
    '        key=(day,t.id,w,freeze)\n'
    '        old=last_record.get(key)\n'
    "        if old is not None and records[old]['end']==a.isoformat(timespec='minutes'):\n"
    "            records[old]['end']=b.isoformat(timespec='minutes'); records[old]['hours']+=hours\n"
    '        else:\n'
    '            last_record[key]=len(records)\n'
    '            records.append(dict(day=day,task=t,workers=w,hours=hours,\n'
    "                                start=a.isoformat(timespec='minutes'),end=b.isoformat(timespec='minutes'),reason=reason))\n"
    '\n'
    '    day=start\n'
    '    while day<=end:\n'
    '        # Schedules share the same shift axis, with a one-hour midday break.\n'
    '        # Shift length is the skilled effective day plus configured overtime.\n'
    "        cal=c.execute('SELECT * FROM calendar WHERE cal_date=?',(day.isoformat(),)).fetchone()\n"
    "        base_ot=float(cal['overtime'] or 0) if cal else 0\n"
    '        curves={}\n'
    '        temp={}\n'
    '        mp=[]\n'
    '        for code,tm in teammap.items():\n'
    '            w,h,ok=capacity.get((code,day),(0,0,False))\n'
    '            if not ok or w<1 or h<=EPS: continue\n'
    "            tc=c.execute('SELECT * FROM team_calendar WHERE cal_date=? AND team=?',(day.isoformat(),code)).fetchone()\n"
    "            ot=max(base_ot,float(tm.get('overtime') or 0),float(tc['overtime'] or 0) if tc else 0)\n"
    "            if scenario.get('ot_hours') is not None: ot=float(scenario['ot_hours'])\n"
    "            if scenario.get('ot_days_per_week') is not None and g['_weekday_work_index'](c,day,g['get_worktime_config'](c))>int(scenario['ot_days_per_week']): ot=0\n"
    "            shift=max(float(tm['sh_per_day'])+ot,h/max(w,1))\n"
    "            if shift>15+EPS: raise ValueError('班组 %s 日班次超过15小时，请先核对作息配置' % code)\n"
    '            curves[code]=(int(w),h,shift,h/(w*shift))\n'
    "            temp[code]=max(0,g['temp_task_daily_hours'](c,code,day,today=start))\n"
    '            for sn,segs in occupy.get(code,{}).items():\n'
    '                for n,nn,sd,ed in segs:\n'
    '                    if (sd and day<sd) or (ed and day>ed): continue\n'
    '                    if n>0: mp.append((code,sn,n,nn))\n'
    '                    break\n'
    '        # Explicit freezes are task-hour quotas. Clamp to updated remaining work.\n'
    '        quotas={}\n'
    '        quota_workers={}\n'
    '        for sn,pc,w,h,z,code in frozen.get(day,[]):\n'
    '            t=keymap.get((sn,pc))\n'
    '            if t and t.remaining>EPS:\n'
    '                quotas[t.id]=min(t.remaining,quotas.get(t.id,0)+h)\n'
    '                quota_workers[t.id]=max(int(w),quota_workers.get(t.id,0),t.min_w)\n'
    '        maxshift=max((x[2] for x in curves.values()),default=0)\n'
    '        for slot in range(math.ceil(maxshift*2)):\n'
    '            offset=slot*.5\n'
    '            wall=8+offset+(1 if offset>=4 else 0)\n'
    '            now=dt.datetime.combine(day,dt.time())+dt.timedelta(hours=wall)\n'
    '            slot_end=now+dt.timedelta(minutes=30)\n'
    '            # 时段级隔离：kinds / zones 每个 30 分钟时段重置一次。\n'
    '            # 工艺冲突（吊装 vs 船下作业等）只要求"同一时段不重叠"，\n'
    '            # 允许同一天先后进行 —— 已与 PMC 确认，是正式口径，勿改回按天互斥。\n'
    '            crew=defaultdict(int); shipcrew=defaultdict(int); zones=defaultdict(set); kinds=[]\n'
    '            for tid,n in parties.items():\n'
    '                held=active[tid]\n'
    '                crew[held.team]+=n\n'
    '                shipcrew[(held.ship_no,held.team)]+=n\n'
    '            slot_h=defaultdict(float)\n'
    '            for code,(w,h,shift,eff) in curves.items():\n'
    '                duration=max(0,min(.5,shift-offset))\n'
    '                if duration<=EPS: continue\n'
    '                # Reserve explicit manpower first, then unnamed temporary work.\n'
    '                for tc,sn,n,nn in mp:\n'
    '                    if tc!=code: continue\n'
    "                    if crew[code]+n>w: raise ValueError('人力占用超过班组人数：%s %s' % (day,code))\n"
    '                    crew[code]+=n\n'
    '                    hrs=n*eff*duration\n'
    '                    used_h[(code,day)]+=hrs\n'
    "                    k=(day,code,sn,'MANPOWER')\n"
    '                    ix=last_record.get(k)\n'
    "                    mp_end=(now+dt.timedelta(hours=duration)).isoformat(timespec='minutes')\n"
    "                    if ix is None or records[ix]['end']!=now.isoformat(timespec='minutes'):\n"
    '                        last_record[k]=len(records)\n'
    "                        records.append(dict(day=day,task=None,ship=sn,team=code,workers=n,hours=hrs,start=now.isoformat(timespec='minutes'),end=mp_end,reason='人力占用模式'))\n"
    "                    else: records[ix]['hours']+=hrs;records[ix]['end']=mp_end\n"
    '                if temp[code]>EPS:\n'
    '                    n=max(0,min(w-crew[code],math.ceil(temp[code]/max(eff*duration,EPS))))\n'
    '                    take=min(temp[code],n*eff*duration)\n'
    '                    crew[code]+=n;used_h[(code,day)]+=take;temp[code]-=take\n'
    '            def priority(t):\n'
    '                due=t.insp or dt.date(2099,12,31)\n'
    '                slack=(due-day).days*8-tail.get(t.id,0)\n'
    '                return (0 if quotas.get(t.id,0)>EPS else 1,0 if t.id in parties else 1,t.prio,\n'
    "                        slack if mode in ('normal_jit','jit') else due.toordinal(),\n"
    '                        -tail.get(t.id,0),t.ship_no,t.code,t.id)\n'
    '            for t in sorted(list(active.values()),key=priority):\n'
    '                if t.remaining<=EPS: continue\n'
    '                frozen_today=quotas.get(t.id,0)>EPS\n'
    '                if (t.ship_no,t.code) in frozen_keys and not frozen_today:\n'
    '                    # Do not pull later frozen work into earlier open time.\n'
    '                    if any(fd>=day and any(x[0]==t.ship_no and x[1]==t.code for x in rs) for fd,rs in frozen.items()): continue\n'
    '                why=ready(t,now)\n'
    '                if why: waits[t.id][why]+=1;continue\n'
    '                curve=curves.get(t.team)\n'
    '                if not curve:\n'
    "                    waits[t.id]['未配置班组 '+t.team if t.team not in teammap else '班组无可用产能或日历停工']+=1\n"
    '                    continue\n'
    '                wtot,htot,shift,eff=curve\n'
    '                duration=max(0,min(.5,shift-offset))\n'
    '                if duration<=EPS: continue\n'
    "                limit=float('inf')\n"
    '                nr=native.get((day,t.ship_no,t.code),[]) if frozen_today else []\n'
    '                if nr:\n'
    "                    live=[r for r in nr if r['planned_start']<=now.isoformat(timespec='minutes')<r['planned_end']]\n"
    '                    if not live: continue\n'
    "                    limit=sum(float(r['planned_hours']) for r in live)\n"
    '                s=shipmap[t.ship_no]; tm=teammap[t.team]\n'
    "                maximum=int(s.get('fixed_workers') or s.get('ship_max_custom') or (tm['ship_max_u'] if s.get('must_finish_on_time') else tm['ship_max']))\n"
    "                if scenario.get('ship_max_override'): maximum=max(maximum,int(scenario['ship_max_override']))\n"
    '                # 理想口径放开【单船并行人数上限】：一条船可同时动用班组的全部人力，\n'
    '                # maximum 取班组总人数 wtot；标准口径仍按 ship_max（初装6/机管4/电装4/甲板5/终装5，紧急船+1）。\n'
    '                # 2026-10-15 用户确认：理想 = 放开单船人数 + 放开舱区班组数，【保留】单工序 4 人。\n'
    "                if mode in ('ideal','jit'): maximum=wtot\n"
    '                held=parties.get(t.id,0)\n'
    '                # 单工序人数上限：两个口径都保留（teams.max_per_task 已在上游取 min 写入 t.max_w）。\n'
    '                cap_w = t.max_w\n'
    '                w=min(cap_w,maximum-shipcrew[(t.ship_no,t.team)]+held,wtot-crew[t.team]+held)\n'
    '                if held:\n'
    '                    if w<held:\n'
    "                        waits[t.id]['原固定作业组产能不足，等待原组恢复；不换人不减员']+=1;continue\n"
    '                    w=held\n'
    '                if frozen_today:\n'
    '                    required=quota_workers[t.id]\n'
    '                    if held and required!=held:\n'
    "                        raise ValueError('冻结计划与固定作业组人数不一致：'+t.ship_no+'/'+t.code+'；请解除旧冻结后重新排程')\n"
    '                    if w<required:\n'
    "                        waits[t.id]['冻结固定作业组人数不足']+=1;continue\n"
    '                    w=required\n'
    '                if w<t.min_w:\n'
    "                    waits[t.id]['%s该船最多只能上%d人（单船并行上限%d人，已被其它工序占%d人），本工序至少要%d人'%(t.team,w,maximum,shipcrew[(t.ship_no,t.team)]+held,t.min_w)]+=1;continue\n"
    '                inzone=zones[(t.ship_no,t.zone)]\n'
    '                zcap=next((v for k,v in zone_cap.items() if k in t.zone),2)\n'
    "                if mode not in ('ideal','jit') and t.zone and t.team not in inzone and len(inzone)>=zcap:\n"
    "                    waits[t.id]['舱区【%s】同时作业班组数已满（上限%d个），%s进不去'%(t.zone,zcap,t.team)]+=1;continue\n"
    '                hit=False\n'
    '                for typ,sn,z in kinds:\n'
    '                    for a,b,scope,level in conflicts:\n'
    "                        if level!='禁止' or not ((t.ttype==a and typ==b) or (t.ttype==b and typ==a)):continue\n"
    "                        if scope=='plant' or (sn==t.ship_no and (scope!='zone' or z==t.zone)): hit=True\n"
    "                if hit: waits[t.id]['%s/%s 与同船或同舱区其他工序工艺互斥（禁止同时作业）'%(t.ship_no,t.code)]+=1;continue\n"
    '                # Fixed new-worker mixture must not inflate team-average output.\n'
    "                if s.get('fixed_workers') and s.get('fixed_new'):\n"
    "                    n=int(s['fixed_workers']); nn=min(n,int(s['fixed_new']))\n"
    "                    mix=((n-nn)*float(tm['sh_per_day'])+nn*float(tm['nh_per_day']))/max(n*float(tm['sh_per_day']),EPS)\n"
    '                    eff=min(eff,mix)\n'
    '                quota=min(t.remaining,quotas.get(t.id,t.remaining),limit)\n'
    '                if not held and not frozen_today:\n'
    '                    w=max(t.min_w,min(w,math.ceil(quota/max(eff*duration,EPS))))\n'
    '                hours=min(quota,w*eff*duration,htot-used_h[(t.team,day)])\n'
    '                if hours<=EPS: continue\n'
    '                parties[t.id]=w\n'
    '                crew[t.team]+=w-held;shipcrew[(t.ship_no,t.team)]+=w-held\n'
    '                inzone.add(t.team);kinds.append((t.ttype,t.ship_no,t.zone))\n'
    '                used_h[(t.team,day)]+=hours\n'
    '                used_w[(t.team,day)]=max(used_w[(t.team,day)],crew[t.team])\n'
    '                b=now+dt.timedelta(hours=duration)\n'
    '                append(t,day,now,b,w,hours,frozen_today)\n'
    '                t.remaining=max(0,t.remaining-hours);t.last_date=day\n'
    '                if frozen_today: quotas[t.id]=max(0,quotas[t.id]-hours)\n'
    '                if t.remaining<=EPS:\n'
    '                    t.finish_date=day;finished[(t.ship_no,t.code)]=day;done_at[(t.ship_no,t.code)]=b;active.pop(t.id,None)\n'
    '                    parties.pop(t.id,None)\n'
    '        bad=[keymap[(t.ship_no,t.code)] for t in tasks if quotas.get(t.id,0)>EPS]\n'
    '        if bad:\n'
    "            detail='；'.join('%s/%s %s' % (t.ship_no,t.code,waits[t.id].most_common(1)[0][0] if waits[t.id] else '冻结工时超出日能力') for t in bad[:6])\n"
    "            raise ValueError('冻结计划无法执行，未发布新版本。请核对放行条件和资源，必要时明确设冻结0天重排：'+detail)\n"
    '        if all(t.remaining<=EPS for t in tasks) and day>=max(frozen,default=start) and not mp:\n'
    '            break\n'
    '        day+=dt.timedelta(days=1)\n'
    '    rows=[];times=[]\n'
    '    for r in records:\n'
    "        t=r['task']\n"
    "        rows.append((None,r['day'].isoformat(),t.ship_no if t else r['ship'],t.ship_type if t else '',t.code if t else 'MANPOWER',t.name if t else '人力占用',t.team if t else r['team'],r['workers'],round(r['hours'],6),t.zone if t else '',t.prio if t else 1,r['reason']))\n"
    "        times.append((r['start'],r['end']))\n"
    '    details=[]\n'
    '    for t in tasks:\n'
    '        if t.remaining<=EPS:continue\n'
    '        why=ready(t,dt.datetime.combine(end,dt.time(23,59)))\n'
    '        cnt=sum(waits[t.id].values())\n'
    "        wc=waits[t.id].most_common(1)[0][0] if waits[t.id] else '展望期内该班组产能/时段不足'\n"
    "        reason=(why+'；卡点：'+wc) if why else wc\n"
    "        if cnt: reason+='（累计 %d 个时段排不进）'%cnt\n"
    '        details.append(dict(ship_no=t.ship_no,process_code=t.code,process_name=t.name,team=t.team,hours=round(t.remaining,4),reason=reason,date=end.isoformat()))\n'
    '    return dict(active=active,results=rows,slot_times=times,used_h=used_h,used_w=used_w,\n'
    '                starved_hours=starved,queued_cnt=queued,blocked=blocked,blocked_detail=details,\n'
    '                precision_details=details)\n'
)
_module = _types.ModuleType('precision_engine')
_module.__file__ = __file__
_sys.modules['precision_engine'] = _module
exec(compile(_code, 'precision_engine', "exec"), _module.__dict__)
_code = (
    '"""Planning readiness and dispatch review UI; local HTTP service only."""\n'
    'import datetime as dt\n'
    'import json\n'
    'import math\n'
    'from pathlib import Path\n'
    '\n'
    'def handle(g,h,q=None,body=None):\n'
    "    c=g['connect']()\n"
    '    try:\n'
    "        g['init_db'](c)\n"
    '        if body is not None:\n'
    "            ident=int(body.get('id',0))\n"
    "            old=c.execute('SELECT * FROM tasks WHERE id=?',(ident,)).fetchone()\n"
    "            if old is None: raise ValueError('工序不存在，请刷新')\n"
    '            fields={}\n'
    "            team=str(body.get('team') or old['team'])\n"
    "            if team!=old['team'] and not c.execute('SELECT 1 FROM teams WHERE team_code=?',(team,)).fetchone():raise ValueError('请选择已配置的班组')\n"
    "            fields['team']=team\n"
    "            for k in ('material_ready','drawing_ready','quality_ready'):\n"
    "                if body.get(k) not in (0,1): raise ValueError('放行状态必须选择就绪或未就绪')\n"
    '                fields[k]=int(body[k])\n'
    "            date=str(body.get('release_date') or '')\n"
    '            if date:dt.date.fromisoformat(date)\n'
    "            fields['release_date']=date or None\n"
    "            for k in ('min_w','max_w'):\n"
    '                v=float(body.get(k,0))\n'
    "                if not math.isfinite(v) or int(v)!=v or not 1<=v<=100:raise ValueError('工序人数应为1至100之间的整数')\n"
    '                fields[k]=int(v)\n'
    "            if fields['max_w']<fields['min_w']:raise ValueError('最多人数不得小于最少人数')\n"
    '            with c:\n'
    "                c.execute('UPDATE tasks SET '+','.join(k+'=?' for k in fields)+' WHERE id=?',tuple(fields.values())+(ident,))\n"
    "                c.execute('INSERT OR REPLACE INTO planning_overrides(ship_no,process_code,payload) VALUES(?,?,?)',\n"
    "                          (old['ship_no'],old['process_code'],json.dumps(fields,ensure_ascii=False)))\n"
    "                c.execute('INSERT INTO readiness_audit(changed_at,task_id,before_json,after_json) VALUES(?,?,?,?)',\n"
    "                          (dt.datetime.now().isoformat(timespec='seconds'),ident,json.dumps({k:old[k] for k in fields},ensure_ascii=False),json.dumps(fields,ensure_ascii=False)))\n"
    "            return h.json({'ok':True,'message':'已保存放行条件；请重新排程以更新计划'})\n"
    "        q=q or {};sn=(q.get('ship') or [''])[0];version=(q.get('version') or [''])[0]\n"
    '        from precision_audit import capture, inspect, status\n'
    "        if (q.get('download') or [''])[0]=='inputs':\n"
    "            saved=c.execute('SELECT * FROM schedule_input_snapshot WHERE version_no=?',(version,)).fetchone()\n"
    "            if saved is None:raise ValueError('该历史版本没有输入快照，请使用新版重新排程')\n"
    "            return h.json(dict(version=version,parameters=json.loads(saved['parameters']),snapshot=json.loads(saved['payload'])))\n"
    "        versions=[dict(r) for r in c.execute('SELECT version_no,created_at,algorithm,is_active,metrics FROM schedule_versions ORDER BY id DESC LIMIT 50')]\n"
    "        if not version:version=next((v['version_no'] for v in versions if v['is_active']), '')\n"
    "        v=c.execute('SELECT * FROM schedule_versions WHERE version_no=?',(version,)).fetchone()\n"
    "        metrics=json.loads(v['metrics'] or '{}') if v else {}\n"
    '        shiprows=[dict(r) for r in c.execute("SELECT ship_no,plan_inspection,plan_start FROM ships WHERE status=\'在制\' ORDER BY ship_no")]\n'
    "        args=[version];condition=''\n"
    "        if sn:condition=' AND ship_no=?';args.append(sn)\n"
    "        rows=[dict(r) for r in c.execute('SELECT * FROM schedule_result WHERE version_no=?'+condition+' ORDER BY planned_start,ship_no,team',args)]\n"
    '        sql="SELECT id,ship_no,process_code,process_name,team,remaining_hours,material_ready,drawing_ready,quality_ready,release_date,min_w,max_w FROM tasks WHERE remaining_hours>0 AND team<>\'前置\' AND ship_no IN (SELECT ship_no FROM ships WHERE status=\'在制\')"\n'
    '        targs=[]\n'
    "        if sn:sql+=' AND ship_no=?';targs=[sn]\n"
    "        ts=[dict(r) for r in c.execute(sql+' ORDER BY ship_no,team,process_code LIMIT 500',targs)]\n"
    "        overrides={(r['ship_no'],r['process_code']):json.loads(r['payload']) for r in c.execute('SELECT * FROM planning_overrides')}\n"
    "        for t in ts:t.update(overrides.get((t['ship_no'],t['process_code']),{}))\n"
    "        details=metrics.get('unscheduled_detail',[])\n"
    "        if sn:details=[d for d in details if d['ship_no']==sn]\n"
    "        snaps=[json.loads(r[0]) for r in c.execute('SELECT payload FROM schedule_ship_snapshot WHERE version_no=?'+condition,args)]\n"
    "        snaps=[s for s in snaps if s.get('status')=='在制']\n"
    "        teams=[r[0] for r in c.execute('SELECT team_code FROM teams ORDER BY sort_order')]\n"
    "        return h.json(dict(input_issues=inspect(capture(c)),stale=status(c,version),version=version,versions=[{k:x[k] for k in ('version_no','created_at','algorithm','is_active')} for x in versions],ships=shiprows,rows=rows,tasks=ts,teams=teams,metrics=metrics,blocked=details,snapshots=snaps))\n"
    '    finally:c.close()\n'
    '\n'
    'HTML=r\'\'\'<!doctype html><html lang="zh-CN"><meta charset="utf-8"><title>精准计划台</title>\n'
    '<style>body{margin:0;background:#f4f6f9;color:#202c3b;font:15px "Microsoft YaHei",sans-serif}main{max-width:1400px;margin:auto;padding:24px}h1{font-size:26px}h2{font-size:19px;margin-top:28px}.card{background:white;border:1px solid #dbe1e8;border-radius:8px;padding:18px;margin:15px 0}.muted{color:#596579;font-size:13px}button,select,input{font:inherit;padding:7px;border:1px solid #b9c7d8;border-radius:5px}button{background:#1859a6;color:white;cursor:pointer}table{border-collapse:collapse;width:100%;font-size:13px}th,td{text-align:left;border:1px solid #dce2e9;padding:8px;white-space:nowrap}th{background:#edf2f8;position:sticky;top:0}.scroll{overflow:auto;max-height:460px}.warn{color:#a84312}#msg{white-space:pre-wrap}dialog{border:1px solid #c3cbd6;border-radius:8px;padding:24px}label{display:block;margin:14px 0}a{color:#1859a6}.row{display:flex;gap:12px;align-items:center;flex-wrap:wrap}</style>\n'
    '<main><a href="/">返回原排产系统</a><h1>精准计划台</h1><p>交期优先 · 30分钟时段 · 放行条件 · 未排原因</p>\n'
    '<div class="card"><div class="row">计划版本 <select id="ver"></select>船舶 <select id="ship"><option value="">全部在制船</option></select><button onclick="load()">刷新</button><button onclick="csv()">下载当前明细 CSV</button></div><p class="muted">正式版本与模拟草稿分别查看。历史版本没有时刻时显示空白，需用升级版重新排程。</p></div>\n'
    '<div class="card"><div class="row">排程起始 <input type="date" id="start">展望天数 <input id="days" type="number" min="1" max="730" value="120" style="width:80px">冻结天数 <input id="freeze" type="number" min="0" max="90" value="0" style="width:70px">排产口径 <select id="pmode"><option value="normal">标准（含舱室上人限制）</option><option value="ideal">理想（放开单船人数+舱区班组数，单工序限4人）</option><option value="normal_jit">标准+JIT</option><option value="jit">理想+JIT</option></select><button onclick="schedule()">生成正式计划</button></div><p class="muted">首次升级默认冻结0天，建立时段基线；后续可选择冻结天数。保存放行条件后需重新排程。此操作会更新本升级版数据库的正式计划。<b>理想口径</b>放开【单船并行人数上限】（初装6/机管4/电装4/甲板5/终装5）与【同舱区同时作业班组数】上限，但<b>单工序人数上限 4 人仍然保留</b>；大节点（船体脱模/甲板/硬顶）与班组总产能同样保留；标准与理想两套计划分别保存，可在【班组计划】按口径出周/月计划。</p></div>\n'
    '<p id="msg" aria-live="polite"></p><div class="card"><b>计划口径</b><p id="summary"></p><p class="warn">本次按前置物料与大件已齐套排总装计划，前置不占产能，也不代表实际完工。时刻按08:00起班、12:00至13:00休息生成；班组采用平均效率，尚未细分实名技能、设备和多个班次。旧数据的图纸与检验默认沿用就绪，必须由计划员核实。理想模式仅供产能上限比较。</p></div>\n'
    '<div class="card"><p id="freshness" class="warn"></p><button onclick="downloadInputs()">下载所选版本输入快照</button></div><h2>当前输入待核对问题</h2><div id="issues" class="scroll"></div><h2>计划交期与完整性</h2><div id="forecast" class="scroll"></div>\n'
    '<h2>未排工序及阻塞原因</h2><div id="blocked" class="scroll"></div>\n'
    '<h2>任务时段明细</h2><div id="rows" class="scroll"></div><p class="muted">页面最多显示2000行；CSV包含当前筛选的全部排程行。同一工序开工后固定作业组人数，跨午休、跨天不换组；最后不足半小时也不减员。分段行是计划作业时段，不是实际打卡记录。尚无实名人员表，作业组不代表具体姓名。</p>\n'
    '<h2>工序放行条件</h2><p class="muted">先按船舶筛选后维护。最多显示500道剩余工序；条件按船号和工序编码保存，重新导入后仍用于排程。</p><div id="tasks" class="scroll"></div>\n'
    '<dialog id="edit"><h2>维护工序开工条件</h2><p id="taskname"></p><label>作业班组 <select id="taskteam"></select></label><label>物料 <select id="material_ready"><option value="1">已齐套</option><option value="0">未齐套</option></select></label><label>图纸 <select id="drawing_ready"><option value="1">已放行</option><option value="0">未放行</option></select></label><label>前置检验 <select id="quality_ready"><option value="1">已放行或不适用</option><option value="0">未放行</option></select></label><label>最早开工日期 <input type="date" id="release_date"></label><label>最少人数 <input type="number" id="min_w" min="1" max="100"></label><label>最多人数 <input type="number" id="max_w" min="1" max="100"></label><button onclick="save()">保存条件</button> <button onclick="document.getElementById(\'edit\').close()">取消</button><p id="editmsg" class="warn"></p></dialog>\n'
    '</main><script>\n'
    'let data={},taskid=null;const $=id=>document.getElementById(id),esc=v=>String(v??\'\').replace(/[&<>"\']/g,c=>({\'&\':\'&amp;\',\'<\':\'&lt;\',\'>\':\'&gt;\',\'"\':\'&quot;\',"\'":\'&#39;\'}[c]));\n'
    'function table(el,cols,rows,edit=false){$(el).innerHTML=\'<table><thead><tr>\'+cols.map(c=>\'<th>\'+esc(c[0])+\'</th>\').join(\'\')+(edit?\'<th>维护</th>\':\'\')+\'</tr></thead><tbody>\'+rows.map(r=>\'<tr>\'+cols.map(c=>\'<td>\'+esc(typeof c[1]===\'function\'?c[1](r):r[c[1]])+\'</td>\').join(\'\')+(edit?\'<td><button onclick="openEdit(\'+Number(r.id)+\')">编辑</button></td>\':\'\')+\'</tr>\').join(\'\')+\'</tbody></table>\';}\n'
    "async function api(url,body){const r=await fetch(url,body?{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(body)}:{});const d=await r.json();if(d.error)throw Error(d.error);return d;}\n"
    'async function load(){try{data=await api(\'/api/precision?ship=\'+encodeURIComponent($(\'ship\').value)+\'&version=\'+encodeURIComponent($(\'ver\').value));const sn=$(\'ship\').value;$(\'ship\').innerHTML=\'<option value="">全部在制船</option>\'+data.ships.map(s=>\'<option>\'+esc(s.ship_no)+\'</option>\').join(\'\');$(\'ship\').value=sn;$(\'ver\').innerHTML=data.versions.map(v=>\'<option value="\'+esc(v.version_no)+\'">\'+esc(v.version_no)+(v.is_active?\' 正式\':\' 草稿或历史\')+\'</option>\').join(\'\');$(\'ver\').value=data.version;render();$(\'msg\').textContent=\'已加载 \'+data.version;}catch(e){$(\'msg\').textContent=e.message;}}\n'
    "function render(){$('freshness').textContent=data.stale===true?'当前数据已改变，所选版本与当前输入不一致，请重新排程。':data.stale===false?'所选版本与当前输入一致；待核对事项仍需现场确认。':'旧版本没有输入快照，无法验证与当前数据的一致性。';table('issues',[['级别','level'],['船号','ship_no'],['工序','process_code'],['问题','message']],(data.input_issues||[]).filter(x=>!$('ship').value||x.ship_no===$('ship').value));const m=data.metrics||{};$('summary').textContent=(m.engine||'旧版排程')+'；剩余未排工序 '+(m.unfinished??'未统计')+' 道；船底门禁拦截 '+(m.keel_blocked_count??0)+' 艘。';table('forecast',[['船号','ship_no'],['计划交检','plan_inspection'],['预测完成','predicted_finish'],['依据','prediction_basis'],['状态',r=>({full:'完整排程',partial:'部分排程',unscheduled:'未排程'}[r.feasibility_status]||r.feasibility_status)],['未排工时','unscheduled_hours'],['风险','risk_level']],data.snapshots);table('blocked',[['船号','ship_no'],['工序','process_code'],['班组','team'],['未排工时','hours'],['原因','reason']],data.blocked);table('rows',[['船号','ship_no'],['工序','process_code'],['名称','process_name'],['班组','team'],['开始','planned_start'],['结束','planned_end'],['人数','workers'],['工时','planned_hours'],['说明','reason']],data.rows.slice(0,2000));table('tasks',[['船号','ship_no'],['工序','process_code'],['名称','process_name'],['班组','team'],['剩余工时','remaining_hours'],['物料',r=>r.material_ready===1?'齐套':'待齐套'],['图纸',r=>r.drawing_ready===1?'就绪':'未放行'],['检验',r=>r.quality_ready===1?'就绪':'未放行'],['最早开工','release_date']],data.tasks,true);}\n"
    "function openEdit(id){const t=data.tasks.find(x=>x.id===id);taskid=id;$('taskteam').innerHTML=[...new Set([t.team,...data.teams])].map(x=>'<option>'+esc(x)+'</option>').join('');$('taskteam').value=t.team;$('taskname').textContent=t.ship_no+' / '+t.process_code+' '+t.process_name;for(const k of ['material_ready','drawing_ready','quality_ready','release_date','min_w','max_w'])$(k).value=t[k]??'';$('editmsg').textContent='';$('edit').showModal();}\n"
    "async function save(){try{let b={id:taskid,team:$('taskteam').value};for(const k of ['material_ready','drawing_ready','quality_ready','min_w','max_w'])b[k]=Number($(k).value);b.release_date=$('release_date').value;await api('/api/precision',b);$('edit').close();await load();$('msg').textContent='条件已保存，请重新排程。';}catch(e){$('editmsg').textContent=e.message;}}\n"
    "async function schedule(){const buttons=[...document.querySelectorAll('button')];try{buttons.forEach(b=>b.disabled=true);$('msg').textContent='正在校验并排程，请稍候…';const r=await api('/api/schedule',{start:$('start').value,horizon:Number($('days').value),freeze:Number($('freeze').value),mode:(document.getElementById('pmode')||{}).value||'normal'});$('ver').innerHTML='<option>'+esc(r.version)+'</option>';await load();}catch(e){$('msg').textContent=e.message;}finally{buttons.forEach(b=>b.disabled=false);}}\n"
    "async function downloadInputs(){try{const d=await api('/api/precision?download=inputs&version='+encodeURIComponent(data.version));const u=URL.createObjectURL(new Blob([JSON.stringify(d,null,2)],{type:'application/json;charset=utf-8'})),a=document.createElement('a');a.href=u;a.download='计划输入_'+data.version+'.json';a.click();URL.revokeObjectURL(u);}catch(e){$('msg').textContent=e.message;}}\n"
    'function csv(){const fields=[\'ship_no\',\'process_code\',\'process_name\',\'team\',\'planned_start\',\'planned_end\',\'workers\',\'planned_hours\',\'reason\'];const quote=x=>\'"\'+String(x??\'\').replace(/^[=+@-]/,"\'$&").replace(/"/g,\'""\')+\'"\';const text=\'\\ufeff\'+[[\'船号\',\'工序\',\'名称\',\'班组\',\'开始\',\'结束\',\'人数\',\'工时\',\'说明\'],...data.rows.map(r=>fields.map(k=>r[k]))].map(r=>r.map(quote).join(\',\')).join(\'\\r\\n\');const u=URL.createObjectURL(new Blob([text],{type:\'text/csv;charset=utf-8\'})),a=document.createElement(\'a\');a.href=u;a.download=\'时段计划_\'+data.version+\'.csv\';a.click();URL.revokeObjectURL(u);}\n'
    "const now=new Date();$('start').value=[now.getFullYear(),String(now.getMonth()+1).padStart(2,'0'),String(now.getDate()).padStart(2,'0')].join('-');load();\n"
    "</script></html>'''\n"
    '\n'
    '\n'
)
_module = _types.ModuleType('precision_web')
_module.__file__ = __file__
_sys.modules['precision_web'] = _module
exec(compile(_code, 'precision_web', "exec"), _module.__dict__)
del _code, _module, _sys, _types


import datetime as dt
import json
import os
import sys
import threading
import traceback
import types
import webbrowser
from collections import defaultdict, Counter, OrderedDict
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import urlparse, parse_qs
import re
import zipfile
import xml.etree.ElementTree as ET
import sqlite3


HTML_UI = r"""<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>船舶总装智能排产系统</title>
<style>
  *{box-sizing:border-box;margin:0;padding:0}
  body{font-family:"Microsoft YaHei","PingFang SC",sans-serif;background:#f4f6f9;color:#1f2937;font-size:14px}
  .top{background:linear-gradient(90deg,#1e3a5f,#2563eb);color:#fff;padding:14px 22px;display:flex;
       align-items:center;justify-content:space-between;box-shadow:0 2px 8px rgba(0,0,0,.15)}
  .top h1{font-size:19px;font-weight:600}
  .top .sub{font-size:12px;opacity:.85;margin-top:3px}
  .status{font-size:12px;background:rgba(255,255,255,.15);padding:6px 12px;border-radius:14px}
  .wrap{display:flex;height:calc(100vh - 62px)}
  .nav{width:158px;background:#fff;border-right:1px solid #e5e7eb;padding:10px 0;overflow-y:auto}
  .nav a{display:block;padding:11px 18px;color:#4b5563;text-decoration:none;font-size:14px;
         border-left:3px solid transparent;cursor:pointer}
  .nav a:hover{background:#f3f4f6}
  .nav a.on{background:#eff6ff;color:#1d4ed8;border-left-color:#2563eb;font-weight:600}
  .main{flex:1;padding:18px 22px;overflow-y:auto}
  h2{font-size:17px;margin-bottom:12px;color:#111827}
  h3{font-size:14px;margin:16px 0 8px;color:#374151}
  .cards{display:flex;gap:12px;flex-wrap:wrap;margin-bottom:16px}
  .card{background:#fff;border-radius:8px;padding:13px 16px;min-width:132px;flex:1;
        box-shadow:0 1px 3px rgba(0,0,0,.08)}
  .card .k{font-size:12px;color:#6b7280}
  .card .v{font-size:23px;font-weight:600;margin-top:4px;color:#111827}
  .card .d{font-size:11px;color:#9ca3af;margin-top:2px}
  .panel{background:#fff;border-radius:8px;padding:16px;margin-bottom:16px;box-shadow:0 1px 3px rgba(0,0,0,.08)}
  table{width:100%;border-collapse:collapse;font-size:13px}
  th{background:#f9fafb;text-align:left;padding:8px 10px;border-bottom:1px solid #e5e7eb;
     color:#374151;font-weight:600;position:sticky;top:0}
  td{padding:7px 10px;border-bottom:1px solid #f3f4f6}
  tr:hover td{background:#f9fafb}
  .tbl-wrap{max-height:520px;overflow:auto;border:1px solid #e5e7eb;border-radius:6px}
  button{background:#2563eb;color:#fff;border:0;padding:8px 16px;border-radius:6px;cursor:pointer;font-size:13px}
  button:hover{background:#1d4ed8}
  button.gray{background:#6b7280}button.gray:hover{background:#4b5563}
  button.green{background:#059669}button.green:hover{background:#047857}
  input,select{padding:6px 9px;border:1px solid #d1d5db;border-radius:5px;font-size:13px}
  .row{display:flex;gap:10px;align-items:center;flex-wrap:wrap;margin-bottom:10px}
  .red{color:#c0392b;font-weight:600}.yellow{color:#b45309;font-weight:600}.green{color:#047857;font-weight:600}
  .tag{display:inline-block;padding:2px 8px;border-radius:10px;font-size:11px;font-weight:600}
  .t-red{background:#fee2e2;color:#b91c1c}.t-yel{background:#fef3c7;color:#92400e}
  .t-grn{background:#d1fae5;color:#065f46}.t-gry{background:#e5e7eb;color:#4b5563}
  .gantt{border:1px solid #e5e7eb;border-radius:6px;overflow:auto;max-height:560px;background:#fff}
  .g-row{display:flex;height:26px;border-bottom:1px solid #f3f4f6;align-items:center}
  .g-label{width:190px;min-width:190px;padding:0 8px;font-size:12px;overflow:hidden;
           white-space:nowrap;text-overflow:ellipsis;border-right:1px solid #e5e7eb;background:#fafafa}
  .g-track{display:flex;position:relative;flex:1}
  .g-cell{width:26px;min-width:26px;height:26px;border-right:1px solid #f0f1f3}
  .g-head{display:flex;height:30px;background:#f9fafb;border-bottom:1px solid #e5e7eb;align-items:center}
  .g-head .g-label{background:#f9fafb;font-weight:600;font-size:12px}
  .g-day{width:26px;min-width:26px;font-size:10px;text-align:center;color:#6b7280}
  .legend{display:flex;gap:12px;font-size:12px;margin:8px 0;flex-wrap:wrap}
  .legend i{display:inline-block;width:12px;height:12px;border-radius:3px;margin-right:4px;vertical-align:-2px}
  .log{background:#111827;color:#d1fae5;font-family:Consolas,monospace;font-size:12px;padding:10px;
       border-radius:6px;max-height:220px;overflow:auto;white-space:pre-wrap;line-height:1.5}
  .msg{padding:9px 12px;border-radius:6px;margin-bottom:10px;font-size:13px}
  .msg.ok{background:#d1fae5;color:#065f46}.msg.err{background:#fee2e2;color:#b91c1c}
  .msg.info{background:#dbeafe;color:#1e40af}
  .muted{color:#6b7280;font-size:12px}
  .loading{display:inline-block;width:14px;height:14px;border:2px solid #fff;border-top-color:transparent;
           border-radius:50%;animation:spin .8s linear infinite;vertical-align:-2px}
  @keyframes spin{to{transform:rotate(360deg)}}
</style>
</head>
<body>
<div class="top">
  <div>
    <h1>船舶总装有限产能智能排产系统</h1>
    <div class="sub">主计划决定先做谁 · 工艺模板决定何时能做 · 人力决定每天能做多少 · 空间冲突决定哪些不能同时做</div>
  </div>
  <div class="status" id="status">加载中…</div>
</div>

<div class="wrap">
  <div class="nav" id="nav">
    <a class="on" data-p="dashboard">📊 总览仪表盘</a>
    <a data-p="import">📥 数据导入</a>
    <a data-p="schedule">🗓 排产与甘特图</a>
    <a data-p="capacity">👷 人力负荷</a>
    <a data-p="risk">⚠️ 风险预警</a>
    <a data-p="estimate">⏱ 交检预测</a>
    <a data-p="teamplan">📅 班组计划</a>
    <a data-p="analysis">📉 产能分析</a>
    <a data-p="scenario">🔀 方案模拟</a>
    <a data-p="settings">⚙️ 系统设置</a>
  </div>
  <div class="main" id="main"></div>
</div>

<script>
const TEAM_COLORS={'初装':'#2563eb','机管':'#0891b2','电装':'#7c3aed','甲板':'#d97706','终装':'#059669'};
const S={page:'dashboard',version:'',data:{}};
let MP_SEGS=[];   // 人力占用的时间段列表（声明放最前，避免 TDZ）
let SHIP_MSG='';  // 船只批量操作的结果提示（重绘后显示一次）
let KEEL_BLOCKED=[];  // 因船底未启动被排除的船（设置页展示用）
let CP_ST={src:'',no:'',type:'',mode:'template',dates:0};  // 复制船面板的输入（整页重绘后恢复）
let CP_SHIPS=[];  // 设置页船只清单快照（复制船取船型用）

function el(id){return document.getElementById(id)}
function esc(v){return String(v??'').replace(/[&<>"']/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]))}
function fmtDate(d){return d?String(d).slice(0,10):''}
function loadColor(r){return r>100?'red':(r>=90?'yellow':'green')}
function riskTag(l){const m={'高风险':'t-red','风险':'t-yel','关注':'t-yel','正常':'t-grn'};
  return '<span class="tag '+(m[l]||'t-gry')+'">'+(l||'-')+'</span>'}

// 排程完整性标签：完整排程 / 部分排程 / 无法排程
function feasiTag(v,unsched){
  const m={'full':['完整排程','t-grn'],'partial':['部分排程','t-yel'],'unscheduled':['无法排程','t-red']};
  if(!v)return '<span class="muted">-</span>';
  const x=m[v]||[v,'t-gry'];
  let t=x[0];
  if(v!=='full'&&unsched>0)t+='<div class="muted" style="font-size:11px">未排'+Math.round(unsched)+'H</div>';
  return '<span class="tag '+x[1]+'">'+t+'</span>';
}

function tbl(rows,cols){
  if(!rows||!rows.length)return '<div class="muted">暂无数据</div>';
  let h='<div class="tbl-wrap"><table><thead><tr>';
  cols.forEach(c=>h+='<th>'+c.t+'</th>');
  h+='</tr></thead><tbody>';
  rows.forEach(r=>{
    h+='<tr>';
    cols.forEach(c=>{
      let v=r[c.k];
      if(c.f)v=c.f(v,r);
      h+='<td>'+(v===undefined||v===null?'':v)+'</td>';
    });
    h+='</tr>';
  });
  return h+'</tbody></table></div>';
}

async function api(url,opt){
  try{
    const r=await fetch(url,opt);
    const j=await r.json();
    if(j&&j.error)throw new Error(j.error);
    return j;
  }catch(e){
    alert('请求失败: '+e.message);
    throw e;
  }
}
function post(url,body){
  return api(url,{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(body||{})});
}

/* ============ 总览 ============ */
async function renderDashboard(){
  el('main').innerHTML='<h2>总览仪表盘</h2><div class="muted">加载中…</div>';
  const d=await api('/api/dashboard');
  S.version=d.version||'';
  let h='<h2>总览仪表盘</h2><div class="cards">';
  h+='<div class="card"><div class="k">在制船数量</div><div class="v">'+d.total+'</div></div>';
  h+='<div class="card"><div class="k">按期船</div><div class="v">'+d.on_time+'</div>'+
     '<div class="d">占比 '+(d.total?(d.on_time/d.total*100).toFixed(0):0)+'%</div></div>';
  h+='<div class="card"><div class="k">风险船</div><div class="v" style="color:#b45309">'+d.risk+'</div></div>';
  h+='<div class="card"><div class="k">预计延期船</div><div class="v" style="color:#c0392b">'+d.delay+'</div>'+
     '<div class="d">合计 '+d.delay_days+' 生产日</div></div>';
  h+='<div class="card"><div class="k">剩余总工时</div><div class="v">'+d.remain.toFixed(0)+
     '<span style="font-size:13px"> H</span></div></div></div>';
  h+='<div class="panel"><h3>班组负荷（未来 7 天）</h3>';
  h+=tbl(d.team_load,[{t:'班组',k:'team'},{t:'人数',k:'workers'},{t:'产能H',k:'capacity'},
    {t:'已排H',k:'planned'},{t:'剩余H',k:'left'},
    {t:'负荷率',k:'load_rate',f:v=>'<span class="'+loadColor(v)+'">'+v.toFixed(0)+'%</span>'}]);
  h+='<div class="muted" style="margin-top:6px">负荷率大于100%红色，90%~100%黄色，低于90%绿色</div></div>';
  h+='<div class="panel"><h3>船只状态</h3>';
  h+=tbl(d.ships,[{t:'船号',k:'ship_no'},{t:'船型',k:'ship_type'},
    {t:'计划交检',k:'plan_inspection',f:fmtDate},{t:'预测交检',k:'predicted_finish',f:fmtDate},
    {t:'延期',k:'delay_days',f:v=>v<0?'<span class="red">排不完</span>':(v>0?'<span class="red">'+v+'</span>':'0')},
    {t:'完成',k:'progress',f:v=>(v*100).toFixed(0)+'%'},
    {t:'剩余H',k:'remaining_hours',f:v=>v.toFixed(0)},
    {t:'阶段',k:'current_stage'},{t:'排程',k:'feasibility_status',
       f:(v,r)=>feasiTag(v,r.unscheduled_hours)},
    {t:'风险',k:'risk_level',f:riskTag},{t:'瓶颈',k:'bottleneck'}]);
  h+='</div>';
  if(S.version)h+='<div class="muted">当前排程版本：'+S.version+'</div>';
  else h+='<div class="msg info">尚未排产，请到【排产与甘特图】点击「开始排产」</div>';
  el('main').innerHTML=h;
}

/* ============ 数据导入 ============ */
async function renderImport(){
  const st=await api('/api/status');
  let rt={files:[],custom_dir:''};
  try{ rt=await api('/api/realtime_files'); }catch(e){}
  let h='<h2>数据导入</h2><div class="panel"><h3>① 原始数据（目录 data/raw）</h3>';
  h+='<div class="row"><button onclick="doCopy()">从桌面复制示例数据</button>'+
     '<button onclick="doImport()">▶ 开始导入（主计划+模板+五班组）</button></div>';
  h+='<div class="muted">已识别：主计划 '+(st.files.master_plan||'无')+' ｜ 船型模板 '+
     st.files.templates.length+' 个 ｜ 班组计划 '+st.files.team_plans.length+' 个</div>';
  h+='<div id="impmsg"></div><div class="log" id="implog">等待操作…</div></div>';

  /* 手动上传：自动扫描找不到时（新表直接放桌面、文件名就是船号）用这个 */
  h+='<div class="panel"><h3>② 手动导入完成情况表（上传 Excel）</h3>'+
     '<div class="row"><input type="file" id="up_files" multiple accept=".xlsx,.xls">'+
     ' <button onclick="doUpload()">⬆ 上传并识别</button></div>'+
     '<div class="muted">自动扫描没找到你新导出的表时（例如直接放在桌面上、文件名就是船号），'+
     '在这里手动选文件，可一次选多个。上传后会自动判断是「单船完成情况表」「船型工艺模板」'+
     '还是「工序工位（工时总表）」并立即导入。</div>'+
     '<div id="upmsg"></div></div>';

  /* 工序工位文件：自动读取五个组的总工时 */
  h+='<div class="panel"><h3>⑥ 工序工位文件（五个组总工时）</h3>'+
     '<div class="row"><input type="file" id="gx_files" multiple accept=".xlsx,.xls">'+
     ' <button onclick="doUploadGongxu()">⬆ 导入工序工位</button>'+
     ' <button class="gray" onclick="loadGongxu()">刷新</button>'+
     ' <button class="gray" onclick="exportGongxu()">⬇ 导出汇总 Excel</button></div>'+
     '<div class="muted">支持直接上传《xxx 工序工位 REVx.xlsx》。程序会读「工时总表」里的'+
     '<b>初装 / 机管（机装+管装）/ 电装 / 甲板 / 终装</b> 五个组的总工时并入库；'+
     '17 个原始班组列的明细也会完整保留。船号只列出来，由你确认后再加入总计划。</div>'+
     '<div id="gxmsg"></div><div id="gxlist"></div></div>';

  /* 扫描目录 */
  h+='<div class="panel"><h3>③ 完成情况表扫描目录</h3><div class="row">'+
     '目录 <input id="rt_dir" style="width:360px" value="'+(rt.custom_dir||'')+
       '" placeholder="留空 = 自动扫描桌面及子文件夹">'+
     ' <button onclick="setRealtimeDir()">💾 保存并重扫</button>'+
     ' <button class="gray" onclick="loadRealtimeFiles()">重新扫描</button>'+
     ' <button class="gray" onclick="importRealtime()">只导入完成情况表</button></div>'+
     '<div class="muted">程序会扫描这里以及桌面上含船号文件的文件夹</div>'+
     '<div id="rtlist"></div></div>';

  h+='<div class="panel"><h3>④ 当前数据库</h3>'+
     '<div class="muted" style="margin-bottom:8px">数据保存在：<b>'+(st.data_dir||'')+
     '</b>（若程序放在U盘/网盘等不可写位置，会自动改存到用户目录）</div><div class="cards">'+
     '<div class="card"><div class="k">船型模板</div><div class="v">'+st.types+'</div></div>'+
     '<div class="card"><div class="k">标准工序</div><div class="v">'+st.templates+'</div></div>'+
     '<div class="card"><div class="k">在制船</div><div class="v">'+st.ships+'</div></div>'+
     '<div class="card"><div class="k">实际工序</div><div class="v">'+st.tasks+'</div></div>'+
     '<div class="card"><div class="k">报工记录</div><div class="v">'+st.progress+'</div></div>'+
     '<div class="card"><div class="k">数据问题</div><div class="v">'+st.quality+'</div></div></div>';
  if(st.types_list&&st.types_list.length){
    h+=tbl(st.types_list,[{t:'船型',k:'type_code'},{t:'周期(生产日)',k:'cycle_days'},
      {t:'标准工时',k:'total_hours'},{t:'工序数',k:'process_count'},{t:'来源',k:'source_file'}]);
  }
  h+='</div><div class="panel"><h3>⑤ 数据质量报告</h3>';
  const q=await api('/api/quality');
  h+=tbl(q,[{t:'严重度',k:'severity'},{t:'来源',k:'src'},{t:'Sheet',k:'sheet'},{t:'行',k:'row_no'},
    {t:'列',k:'col'},{t:'问题',k:'issue'},{t:'说明',k:'detail'},{t:'原值',k:'raw'}]);
  h+='<div class="muted" style="margin-top:6px">任何单元格异常都不会中断导入，只会记录在这里</div></div>';
  el('main').innerHTML=h;
  try{ await loadGongxu(); }catch(e){}
}
async function doCopy(){
  el('implog').textContent='复制中…';
  const r=await post('/api/copy');
  el('implog').textContent=r.copied.length
    ? ('已同步 '+r.copied.length+' 个文件到 data/raw：\n'+r.copied.join('\n'))
    : '桌面上没有比 data/raw 更新的同名文件（已是最新），或桌面没找到这类文件。\n'+
      '说明：程序只从 data/raw 读取；把新的班组周计划按同名放到桌面后，'+
      '这里会按修改时间自动覆盖。';
  el('impmsg').innerHTML='<div class="msg ok">复制完成，请点击「开始导入」</div>';
}
/* ---- 手动上传 / 扫描目录 ---- */
const KIND_CN={realtime:'单船完成情况表',template:'船型工艺模板',master:'主计划',gongxu:'工序工位（工时总表）',plan:'排布计划详情',unknown:'未识别'};
function renderRealtimeList(rt){
  const box=el('rtlist');
  if(!box)return;
  if(!rt.files || !rt.files.length){
    box.innerHTML='<div class="msg err">没有扫描到完成情况表。请手动指定目录，或用上面的上传按钮。</div>';
    return;
  }
  let h='<div class="muted" style="margin-bottom:6px">共识别 '+rt.files.length+' 个：</div>';
  h+=tbl(rt.files,[{t:'文件',k:'name'},
    {t:'识别类型',k:'kind',f:v=>KIND_CN[v]||v},
    {t:'所在目录',k:'path',f:v=>v?v.replace(/[^\\\/]+$/,''):''}]);
  box.innerHTML=h;
}
async function loadRealtimeFiles(){
  const rt=await api('/api/realtime_files');
  renderRealtimeList(rt);
}
async function setRealtimeDir(){
  await post('/api/realtime_files',{act:'setdir',path:el('rt_dir').value||''});
  await loadRealtimeFiles();
}
async function importRealtime(){
  el('rtlist').innerHTML='<div class="msg info">导入中…</div>';
  const r=await post('/api/import/realtime',{});
  let h='<div class="msg ok">已导入 '+r.imported.length+' 个文件；交叉核对：'+
    '完成 '+(r.cross.done||0)+' 道、修正剩余 '+(r.cross.remain_fixed||0)+' 道、补建 '+(r.cross.created||0)+' 道</div>';
  for(const x of r.imported){
    h+='<div class="muted">'+x.file+'：'+(x.records!==undefined?(x.records+' 道，完成 '+(x.done||0)):(x.error||''))+'</div>';
  }
  el('rtlist').innerHTML=h;
}
async function doUpload(){
  const inp=el('up_files');
  const files=inp&&inp.files?Array.from(inp.files):[];
  if(!files.length){alert('请先点“选择文件”挑一个 Excel');return}
  el('upmsg').innerHTML='<div class="msg info">上传中…（共 '+files.length+' 个）</div>';
  const outs=[];
  for(const f of files){
    try{
      const b64=await new Promise((res,rej)=>{
        const r=new FileReader();
        r.onload=()=>res(r.result);
        r.onerror=()=>rej(new Error('读取失败'));
        r.readAsDataURL(f);
      });
      outs.push(await post('/api/upload',{name:f.name, data:String(b64).split(',')[1]}));
    }catch(e){ outs.push({file:f.name, error:e.message}); }
  }
  let h='';
  outs.forEach(o=>{
    if(o.error) h+='<div class="msg err">'+o.file+'：'+o.error+'</div>';
    else{
      const r=o.result||{};
      h+='<div class="msg ok">'+o.file+' → 识别为【'+KIND_CN[o.kind]+
         '】'+(r.records!==undefined?('，导入 '+r.records+' 道，完成 '+(r.done||0)+' 道'):'')+'</div>';
    }
  });
  h+='<div class="muted">上传的文件已存到 data/raw；若要让全量流程重算，请点上面的「开始导入」</div>';
  el('upmsg').innerHTML=h;
  try{ await loadRealtimeFiles(); }catch(e){}
}

/* ---- 工序工位：五个组总工时 ---- */
function _readB64(f){
  return new Promise((res,rej)=>{const r=new FileReader();
    r.onload=()=>res(String(r.result).split(',')[1]);r.onerror=()=>rej(new Error('读取失败'));
    r.readAsDataURL(f);});
}
async function doUploadGongxu(){
  const inp=el('gx_files');
  const files=inp&&inp.files?Array.from(inp.files):[];
  if(!files.length){alert('请先点“选择文件”挑一个工序工位 Excel');return}
  el('gxmsg').innerHTML='<div class="msg info">解析中…（共 '+files.length+' 个）</div>';
  let h='';
  for(const f of files){
    try{
      const b64=await _readB64(f);
      const o=await post('/api/upload',{name:f.name,data:b64});
      if(o.error){ h+='<div class="msg err">'+o.file+'：'+o.error+'</div>'; continue; }
      const r=o.result||{};
      const kd=KIND_CN[o.kind]||o.kind;
      if(r.rows!==undefined){
        h+='<div class="msg ok">'+o.file+' → 识别为【'+kd+'】'+
           (r.rev?(' '+r.rev):'')+'：解析船型行 '+r.rows+' 条，写入五组工时 '+r.hours+' 条</div>';
        (r.parsed&&r.parsed.rows||[]).forEach(x=>{
          const g=x.groups||{};
          h+='<div class="muted" style="padding-left:14px">　船型 '+esc(x.ship_type)+
             (x.ship_no?('　船号 '+esc(x.ship_no)):'')+'：'+
             Object.keys(g).map(k=>k+' '+g[k]).join(' ｜ ')+'　合计 '+x.calc_total+'</div>';
        });
      }else{
        h+='<div class="msg ok">'+o.file+' → 识别为【'+kd+'】'+
           (r.records!==undefined?('，导入 '+r.records+' 道，完成 '+(r.done||0)+' 道'):'')+'</div>';
      }
    }catch(e){ h+='<div class="msg err">'+esc(f.name)+'：'+esc(e.message)+'</div>'; }
  }
  el('gxmsg').innerHTML=h;
  await loadGongxu();
}
async function loadGongxu(){
  const box=el('gxlist'); if(!box)return;
  let d; try{ d=await api('/api/gongxu'); }catch(e){ box.innerHTML='<div class="msg err">'+e.message+'</div>'; return; }
  const gs=d.groups||[];
  let h='';
  if(d.rows&&d.rows.length){
    h+='<div class="muted" style="margin:8px 0 6px">已入库：'+d.rows.length+' 艘船的五组总工时（单位：小时）</div>';
    const cols=[{t:'船号',k:'ship_no'},{t:'船型',k:'ship_type'}]
      .concat(gs.map(g=>({t:g,k:'__gx__',
        f:(v,row)=>{const x=(row.groups||{})[g];return (x===undefined?'-':x);}})))
      .concat([{t:'五组合计',k:'total'},
               {t:'来源文件',k:'src_file',f:v=>(v||'').slice(0,26)}]);
    h+=tbl(d.rows,cols);
  }else{
    h+='<div class="msg info">还没有工序工位数据。用上面的按钮上传《xxx 工序工位 REVx.xlsx》即可。</div>';
  }
  const pend=(d.stages||[]).filter(s=>!s.exists);
  if(pend.length){
    h+='<div style="margin-top:14px"><b>待确认是否加入总计划</b>'+
       '<span class="muted">（识别到船号，但总计划里还没有这艘船）</span></div>'+
       '<div class="muted" style="margin:6px 0">船号 '+
       pend.map(s=>esc(s.ship_no)+(s.ship_type?('('+esc(s.ship_type)+')'):'')).join('、')+'</div>'+
       '<div class="row"><button onclick="addGongxuShips()">➕ 把以上船号加入总计划</button></div>'+
       '<div class="muted">加入后其余时间节点（船底启动/脱模/交检等）为空，请在【总计划】页补填。</div>';
  }
  const all=(d.stages||[]);
  if(all.length){
    h+='<div style="margin-top:14px"><b>工序工位文件台账</b></div>';
    h+=tbl(all,[{t:'船号',k:'ship_no'},{t:'船型',k:'ship_type'},{t:'版本',k:'rev'},
      {t:'表内合计',k:'total_hours'},{t:'来源文件',k:'src_file',f:v=>(v||'').slice(0,30)},
      {t:'已加总计划',k:'added',f:v=>v?'是':'否'},{t:'导入时间',k:'imported_at'}]);
  }
  box.innerHTML=h;
}
async function addGongxuShips(){
  const d=await api('/api/gongxu');
  const nos=(d.stages||[]).filter(s=>!s.exists).map(s=>s.ship_no);
  if(!nos.length){alert('没有待加入的船号');return}
  if(!confirm('确认把这 '+nos.length+' 个船号加入总计划？\n\n'+nos.join('、')+
     '\n\n加入后时间节点为空，需要你在【总计划】页补填。'))return;
  const r=await post('/api/gongxu',{act:'add',ship_nos:nos});
  el('gxmsg').innerHTML='<div class="msg ok">已加入 '+r.added.length+' 艘：'+
    (r.added.join('、')||'无')+(r.existed.length?('；已存在 '+r.existed.length+' 艘'):'')+'</div>';
  await loadGongxu();
}
async function exportGongxu(){
  const d=await api('/api/gongxu');
  const gs=d.groups||[];
  const head=['船号','船型'].concat(gs).concat(['五组合计','来源文件','版本']);
  const lines=[head.join(',')];
  (d.rows||[]).forEach(r=>{
    const row=[r.ship_no,r.ship_type].concat(gs.map(g=>{
      const v=(r.groups||{})[g]; return v===undefined?'':v;})).concat([r.total,r.src_file||'',r.rev||'']);
    lines.push(row.map(v=>'"'+String(v).replace(/"/g,'""')+'"').join(','));
  });
  const bom='\ufeff';
  const u=URL.createObjectURL(new Blob([bom+lines.join('\r\n')],{type:'text/csv;charset=utf-8'}));
  const a=document.createElement('a');a.href=u;a.download='工序工位_五组总工时.csv';a.click();
  URL.revokeObjectURL(u);
}

async function doImport(){
  el('implog').textContent='导入中（约 10 秒）…';  el('impmsg').innerHTML='<div class="msg info">正在导入并清洗数据…</div>';
  const r=await post('/api/import');
  el('implog').textContent=(r.logs||[]).join('\n');
  let ok='导入完成：在制船 '+r.stats.in_production_ships+' 艘';
  const cr=r.stats&&r.stats.cross, op=r.stats&&r.stats.option_skipped;
  const un=(r.stats&&r.stats.unstarted_teams)||[], ts=r.stats&&r.stats.team_scope_skipped;
  const fr=r.stats&&r.stats.final_remaining;
  if(cr) ok+=' ｜ 交叉核对判定完成 '+cr.done+' 道（班组情况表 '+cr.from_team_plan+
    ' + 完成情况表 '+cr.from_realtime+
    (cr.from_plan_detail?(' + 排布单 '+cr.from_plan_detail):'')+'）'+
    (cr.team_fixed_conflict?('，其中 <b>'+cr.team_fixed_conflict+'</b> 道靠班组情况表补齐'):'');
  if(un.length) ok+=' ｜ <b>'+un.length+'</b> 个班组未启动，按 100% 满额计入';
  if(ts&&ts.tasks) ok+=' ｜ 班组表未列工序 '+ts.tasks+' 道不计入';
  if(op&&op.tasks) ok+=' ｜ 选配(O/0) '+op.tasks+' 道不计入';
  if(fr) ok+=' ｜ <b>最终剩余工时 '+fr.hours+' H</b>（'+fr.kept+' 道计入、'+
    fr.dropped+' 道剔除完成/选配，两表工时取小值）';
  el('impmsg').innerHTML='<div class="msg ok">'+ok+'</div>';
  if(un.length){
    let t='<div class="muted" style="margin-top:6px">未启动班组（按满额）：'+
      un.slice(0,10).map(x=>x.ship_no+'·'+x.team+' '+x.hours+'H').join('、')+
      (un.length>10?' …':'')+'</div>';
    el('impmsg').innerHTML+=t;
  }
  loadStatus();
}

/* ============ 排产 ============ */
async function renderSchedule(){
  const st=await api('/api/status');
  let h='<h2>排产与甘特图</h2>';
  h+='<div class="panel"><div class="row">起始日 <input type="date" id="sd" value="'+st.today+'">'+
     ' 展望期(天) <input type="number" id="hz" value="120" style="width:80px">'+
     ' 冻结天数 <input type="number" id="fz" value="0" style="width:70px">'+
     ' 模型 <select id="schmode">'+
     '<option value="normal">标准（受单船人数/区域约束，尽快做完）</option>'+
     '<option value="ideal">理想（放开单船人数+舱区班组数；单工序仍限4人）</option>'+
     '<option value="jit">理想 + 准时制 JIT（同上口径，但卡在交检日完成）</option>'+
     '<option value="normal_jit">标准 + 准时制 JIT（受单船人数/区域约束，但按紧迫度让路）</option></select>'+
     ' <button onclick="doSchedule()">▶ 开始排产</button>'+
     ' <button class="green" onclick="doExport()">导出 Excel</button>'+
     ' <label style="display:inline-flex;align-items:center;gap:4px;margin-left:6px">'+
       '<input type="checkbox" id="sch_nokeel"> <b>本次无视船底门禁</b></label></div>'+
     '<div class="muted">冻结窗口内已发布的计划不会被自动修改。'+
     '勾选「本次无视船底门禁」= 这一次排产把船底未启动的船也照排进去'+
     '（只影响本次，不改系统设置）<br>'+
     '<b>口径区别</b>：<b>标准</b>= 考虑舱室上人限制（单船并行人数上限 + 同舱区同时作业班组数上限）；'+
     '<b>理想</b>= 不考虑上人限制，只保留船体脱模/甲板/硬顶等大节点（模板开工日 + 工序前后置）'+
     '与班组产能约束，用来测算"理论最快"能做多快。两套计划互不覆盖，'+
     '可在【班组计划】里按口径分别出周计划、月计划。</div>'+
     '<div id="schmsg"></div></div>';
  // 船底启动门禁状态提示（排产前就要看得见）
  try{
    const kg=await api('/api/keel_gate');
    if(kg.keel_gate){
      const _rn={auto:'实际优先，计划已到也算',plan:'只看计划船底启动日',
                 sale:'只看实际船底启动日'}[kg.keel_rule]||kg.keel_rule;
      h+='<div class="panel" style="background:#fff8e6;border:1px solid #f0d9a0">'+
        '<b>船底启动门禁已开启</b>（'+_rn+'）：'+
        (kg.blocked_count
          ? '本次排产将排除 <b class="t-red" style="padding:0 4px">'+kg.blocked_count+
            '</b> 艘船底未启动的船 —— '+
            kg.blocked.slice(0,15).map(b=>b.ship_no).join('、')+
            (kg.blocked_count>15?' 等':'')
          : '当前没有被排除的船 ✔')+
        '<div class="muted" style="margin-top:4px">口径与开关在【系统设置 → ② 船只优先级与锁定】里调整</div></div>';
    }
  }catch(e){}
  if(S.version){
    const d=await api('/api/ships');
    h+='<div class="panel"><h3>排程结果</h3><div class="cards">'+
      '<div class="card"><div class="k">版本号</div><div class="v" style="font-size:16px">'+S.version+'</div></div>'+
      '<div class="card"><div class="k">排程明细</div><div class="v">'+d.rows+'</div><div class="d">条日计划</div></div>'+
      '<div class="card"><div class="k">延期船</div><div class="v" style="color:#c0392b">'+d.delay_ships.length+'</div></div>'+
      '<div class="card"><div class="k">总延期</div><div class="v">'+d.total_delay+'</div><div class="d">生产日</div></div>'+
      '<div class="card"><div class="k">未完成工序</div><div class="v">'+d.unfinished+'</div></div></div>';
    h+=tbl(d.ships,[{t:'船号',k:'ship_no',f:(v,r)=>v+(r.overdue?' <span class="red">[已逾期]</span>':'')},
      {t:'船型',k:'ship_type'},
      {t:'计划交检',k:'plan_inspection',f:fmtDate},{t:'预测交检',k:'predicted_finish',f:fmtDate},
      {t:'延期',k:'delay_days',f:v=>v<0?'<span class="red">排不完</span>':(v>0?'<span class="red">'+v+'</span>':'<span class="green">0</span>')},
      {t:'排程',k:'feasibility_status',f:(v,r)=>feasiTag(v,r.unscheduled_hours)},
      {t:'风险',k:'risk_level',f:riskTag},{t:'瓶颈',k:'bottleneck'},
      {t:'原因',k:'delay_reason',f:v=>String(v||'').slice(0,70)}]);
    h+='</div>';
    h+='<div class="panel"><h3>甘特图</h3><div class="row">视图 '+
      '<select id="gmode" onchange="fillGanttKey()"><option value="ship">船只视图</option>'+
      '<option value="team">班组视图</option></select><span id="gkey"></span>'+
      ' 显示天数 <input type="number" id="gdays" value="60" style="width:70px" onchange="drawGantt()">'+
      '</div><div id="gantt"></div></div>';
  }
  el('main').innerHTML=h;
  if(S.version){await fillGanttKey();await drawGantt();}
}
async function fillGanttKey(){
  const d=await api('/api/ships');
  const m=el('gmode').value;
  const box=el('gkey');
  if(m==='ship'){
    box.innerHTML=' 船号 <select id="gsel" onchange="drawGantt()">'+
      d.ships.map(s=>'<option value="'+s.ship_no+'">'+s.ship_no+'</option>').join('')+'</select>';
  }else{
    box.innerHTML=' 班组 <select id="gsel" onchange="drawGantt()">'+
      ['初装','机管','电装','甲板','终装'].map(t=>'<option value="'+t+'">'+t+'</option>').join('')+'</select>';
  }
  await drawGantt();
}
async function drawGantt(){
  const mode=el('gmode')?el('gmode').value:'ship';
  const key=el('gsel')?el('gsel').value:'';
  const days=el('gdays')?(el('gdays').value||60):60;
  if(!key)return;
  el('gantt').innerHTML='<div class="muted">加载中…</div>';
  const g=await api('/api/gantt?mode='+mode+'&key='+encodeURIComponent(key)+'&days='+days);
  if(!g.rows.length){el('gantt').innerHTML='<div class="muted">该对象在本次排程中没有安排</div>';return}
  let h='<div class="gantt"><div class="g-head"><div class="g-label">'+
        (mode==='ship'?'工序':'船号')+'</div>';
  g.dates.forEach(d=>{
    const wk=(d.wk>=6);
    h+='<div class="g-day" style="'+(wk?'color:#dc2626':'')+'">'+d.d+'</div>';
  });
  h+='</div>';
  g.rows.forEach(r=>{
    h+='<div class="g-row"><div class="g-label" title="'+r.label+'">'+r.label+'</div><div class="g-track">';
    g.dates.forEach((d,i)=>{
      const c=r.cells[i];
      let bg='',title='';
      if(c){bg=TEAM_COLORS[c.team]||'#2563eb';
        title=d.full+' '+(c.name||'')+' '+c.w+'人 '+c.h+'H '+(c.zone||'');}
      else if(d.wk>=6)bg='#f5f5f4';
      h+='<div class="g-cell" style="'+(bg?'background:'+bg:'')+'" title="'+title+'"></div>';
    });
    h+='</div></div>';
  });
  h+='</div><div class="legend">';
  Object.keys(TEAM_COLORS).forEach(k=>{
    h+='<span><i style="background:'+TEAM_COLORS[k]+'"></i>'+k+'</span>';});
  h+='<span><i style="background:#f5f5f4;border:1px solid #e5e7eb"></i>周末/非生产日</span>'+
     '<span class="muted">鼠标悬停查看当天安排的人数与工时</span></div>';
  el('gantt').innerHTML=h;
}
async function doSchedule(){
  const body={start:el('sd').value,horizon:+el('hz').value,freeze:+el('fz').value,
    mode:(el('schmode')&&el('schmode').value)||'normal'};
  // 「本次无视船底门禁」：只对这一次排产生效，不动系统设置
  if(el('sch_nokeel')&&el('sch_nokeel').checked) body.keel_gate=0;
  const mn={normal:'标准模型',ideal:'理想模型',jit:'理想+JIT',normal_jit:'标准+JIT'}[body.mode]||'';
  el('schmsg').innerHTML='<div class="msg info">正在排产…'+(mn?'（'+mn+'）':'')+
    (body.keel_gate===0?'（本次无视船底门禁）':'')+'</div>';
  const r=await post('/api/schedule',body);
  if(r.error){ el('schmsg').innerHTML='<div class="msg err">排产失败：'+esc(r.error)+'</div>'; return; }
  S.version=r.version;
  let ok='排产完成，版本 '+r.version+'：安排 '+r.metrics.rows+' 条，延期船 '+
    r.metrics.delay_ships.length+' 艘';
  const kb=r.metrics.keel_blocked_count||0;
  if(kb){
    ok+='。因<b>船底未启动</b>排除 '+kb+' 艘：'+
      (r.metrics.keel_blocked||[]).slice(0,8).map(b=>b.ship_no).join('、')+
      (kb>8?' 等 '+kb+' 艘':'');
  }
  el('schmsg').innerHTML='<div class="msg ok">'+ok+'</div>';
  await renderSchedule();loadStatus();
}
async function doExport(){
  const r=await post('/api/export',{version:S.version});
  alert('已导出：\n'+r.path+'\n\n（5 个工作表：主计划预测 / 每日排产 / 班组日计划 / 人力负荷 / 风险预警）');
}

/* ============ 人力负荷 ============ */
async function renderCapacity(){
  const st=await api('/api/status');
  let h='<h2>人力负荷</h2><div class="panel"><div class="row">起始日 '+
    '<input type="date" id="cd" value="'+st.today+'"> 天数 '+
    '<input type="number" id="cdays" value="30" style="width:80px">'+
    ' <button onclick="loadCapacity()">查询</button></div>'+
    '<div class="muted">可用工时 = 熟练工×有效工时 + 新人×有效工时（不是总人数×熟练工工时）</div></div>'+
    '<div class="panel"><h3>船期延期设置</h3>'+
    '<div class="muted">勾选一条或多条船 → 填延期天数（可为负=提前）→ 保存。保存后<b>重新排产</b>才生效：'+
    '该船的船底启动/脱模/甲板/硬顶/交检日期会整体顺延，班组周月计划随之变化。延期天数填 0 即取消。</div>'+
    '<div class="row"><b>已选船</b> <span id="dlSel">（未选）</span> '+
    '<button class="green" onclick="dlSave()">保存延期</button> '+
    '<button onclick="dlClear()">清零延期</button> '+
    '<button onclick="dlAllClear()">全部清零</button></div>'+
    '<div class="row">统一延期天数 <input type="number" id="dlDays" value="7" style="width:80px"> 天　'+
    '原因 <input type="text" id="dlReason" placeholder="如：客户要求推迟交检 / 齐套件未到" '+
    'style="width:280px"></div>'+
    '<div id="dlBody"></div></div>'+
    '<div id="capbody"></div>';
  el('main').innerHTML=h;
  await loadDelayForm();
  await loadCapacity();
}
/* ===== 船期延期表单 ===== */
const DL_SEL={};
async function loadDelayForm(){
  const d=await post('/api/ship/delay',{act:'list'});
  DL_ROWS=d.rows||[];
  let h=tbl(DL_ROWS,[{t:'',k:'_x',f:(_,r)=>DL_CHK(r.ship_no)},
    {t:'船号',k:'ship_no'},{t:'船型',k:'ship_type'},
    {t:'计划交检',k:'plan_inspection'},{t:'顺延后交检',k:'delayed_inspection',
     f:(v,r)=>r.delay_days>0?'<span class="red">'+v+'（+'+r.delay_days+'天）</span>':v},
    {t:'预测完成',k:'predicted_finish'},
    {t:'剩余H',k:'remaining_hours'},{t:'瓶颈班组',k:'bottleneck'},
    {t:'已设延期',k:'delay_days',f:v=>v>0?'<span class="red">延'+v+'天</span>':(v<0?'提前'+(-v)+'天':'—')},
    {t:'原因',k:'reason'}],true);
  el('dlBody').innerHTML=h;
}
function DL_CHK(sn){
  DL_SEL[sn]=(DL_SEL[sn]||false);
  return '<input type="checkbox" '+(DL_SEL[sn]?'checked':'')+
    ' onchange="DL_SEL[\''+sn+'\']=this.checked;dlRefresh()">';
}
function dlRefresh(){
  const s=Object.keys(DL_SEL).filter(k=>DL_SEL[k]);
  el('dlSel').innerHTML=s.length?('<b>'+s.join('、')+'</b>（'+s.length+' 艘）'):'（未选）';
}
async function dlSave(){
  const ships=Object.keys(DL_SEL).filter(k=>DL_SEL[k]);
  if(!ships.length){alert('请先勾选要设置延期的船（表左侧第一列）');return;}
  const days=+el('dlDays').value||0, reason=el('dlReason').value.trim();
  if(!confirm('把 '+ships.length+' 艘船顺延 '+days+' 天？\n'+ships.join('、')+'\n\n保存后需要重新排产才生效。'))return;
  const r=await post('/api/ship/delay',{act:'set',ships:ships,delay_days:days,reason:reason});
  if(r.error){alert(r.error);return;}
  DL_ROWS.forEach(x=>DL_SEL[x.ship_no]=false); await loadDelayForm();
  alert('已保存延期：\n'+(r.saved||[]).join('、')+'\n统一顺延 '+r.delay_days+' 天。\n\n请到【排产与甘特图】重新排产后生效。');
}
async function dlClear(){
  const ships=Object.keys(DL_SEL).filter(k=>DL_SEL[k]);
  if(!ships.length){alert('请先勾选船');return;}
  await post('/api/ship/delay',{act:'clear',ships:ships});
  DL_ROWS.forEach(x=>DL_SEL[x.ship_no]=false); await loadDelayForm();
  alert('已取消这 '+ships.length+' 艘船的延期设置，重新排产后恢复原计划。');
}
async function dlAllClear(){
  if(!confirm('清空【所有船】的延期设置？'))return;
  const ships=DL_ROWS.map(x=>x.ship_no);
  if(!ships.length){alert('没有在制船');return;}
  const r=await post('/api/ship/delay',{act:'clear',ships:ships});
  DL_ROWS.forEach(x=>DL_SEL[x.ship_no]=false); await loadDelayForm();
  alert('已清空 '+ships.length+' 艘船的延期设置，重新排产后生效。');
}
async function loadCapacity(){
  const start=el('cd').value,days=+el('cdays').value;
  el('capbody').innerHTML='<div class="muted">计算中…</div>';
  const d=await api('/api/capacity?start='+start+'&days='+days);
  let h='<div class="panel"><h3>班组周期汇总</h3>';
  h+=tbl(d.summary,[{t:'班组',k:'team'},{t:'可用H',k:'capacity'},{t:'计划H',k:'planned'},
    {t:'剩余H',k:'left'},{t:'负荷率',k:'load_rate',f:v=>'<span class="'+loadColor(v)+'">'+v.toFixed(0)+'%</span>'},
    {t:'超载天数',k:'over_days',f:v=>v>0?'<span class="red">'+v+'</span>':'0'}]);
  h+='</div><div class="panel"><h3>每日负荷曲线</h3><div id="chart"></div></div>'+
     '<div class="panel"><h3>明细</h3>';
  h+=tbl(d.rows,[{t:'日期',k:'date'},{t:'星期',k:'wk'},{t:'班组',k:'team'},
    {t:'生产日',k:'prod'},{t:'可用人数',k:'workers'},{t:'可用H',k:'capacity'},{t:'计划H',k:'planned'},
    {t:'剩余H',k:'left'},{t:'负荷率',k:'rate',f:v=>v>0?'<span class="'+loadColor(v)+'">'+v.toFixed(0)+'%</span>':'-'},
    {t:'超载H',k:'over',f:v=>v>0?'<span class="red">'+v.toFixed(1)+'</span>':'0'}]);
  h+='</div>';
  el('capbody').innerHTML=h;
  drawChart(d.rows,d.dates);
}
function drawChart(rows,dates){
  const teams=[...new Set(rows.map(r=>r.team))];
  const W=Math.max(600,dates.length*22),H=230,pad=28;
  const maxY=Math.max(120,...rows.map(r=>r.rate));
  const colors=['#2563eb','#0891b2','#7c3aed','#d97706','#059669'];
  let s='<svg width="100%" viewBox="0 0 '+W+' '+H+'" style="max-width:100%;height:'+H+'px">';
  [50,100,150].forEach(v=>{
    if(v>maxY)return;
    const y=H-pad-(v/maxY)*(H-pad*2);
    s+='<line x1="'+pad+'" y1="'+y+'" x2="'+(W-10)+'" y2="'+y+'" stroke="#e5e7eb"/>'+
       '<text x="4" y="'+(y+4)+'" font-size="10" fill="#9ca3af">'+v+'%</text>';
  });
  teams.forEach((t,ti)=>{
    const pts=dates.map((d,i)=>{
      const r=rows.find(x=>x.team===t&&x.date===d);
      const v=r?r.rate:0;
      return (pad+i*((W-pad)/dates.length))+','+(H-pad-(v/maxY)*(H-pad*2));
    });
    s+='<polyline points="'+pts.join(' ')+'" fill="none" stroke="'+colors[ti%5]+'" stroke-width="2"/>';
  });
  const y100=H-pad-(100/maxY)*(H-pad*2);
  s+='<line x1="'+pad+'" y1="'+y100+'" x2="'+(W-10)+'" y2="'+y100+'" stroke="#ef4444" stroke-dasharray="4"/></svg>';
  let lg='';
  teams.forEach((t,i)=>lg+='<span style="color:'+colors[i%5]+'">■ '+t+'</span>&nbsp;&nbsp;');
  el('chart').innerHTML=s+'<div class="legend">'+lg+
    '<span class="muted">红色虚线 = 100% 超载线</span></div>';
}

/* ============ 风险 ============ */
async function renderRisk(){
  const d=await api('/api/risk');
  let h='<h2>风险预警与调度建议</h2><div class="panel"><h3>风险清单</h3>';
  h+=tbl(d.risks,[{t:'类型',k:'类型'},{t:'对象',k:'对象'},
    {t:'等级',k:'等级',f:v=>'<span class="tag '+(v==='高'?'t-red':(v==='中'?'t-yel':'t-gry'))+'">'+v+'</span>'},
    {t:'时间',k:'时间'},{t:'缺工时H',k:'缺工时H',f:v=>(v>0?'<span class="red">'+v+'</span>':'—')},
    {t:'缺人数',k:'缺人数',f:v=>(v>0?'<span class="red">补'+v+'人</span>':'—')},
    {t:'描述',k:'描述'},{t:'原因',k:'原因'},{t:'建议',k:'建议'}]);
  h+='</div><div class="panel"><h3>单船调度建议</h3><div class="row">船号 '+
     '<select id="rship">'+d.ships.map(s=>'<option>'+s+'</option>').join('')+'</select>'+
     ' <button onclick="loadSuggest()">生成建议</button></div><div id="sug"></div></div>';
  el('main').innerHTML=h;
}
async function loadSuggest(){
  const ship=el('rship').value;
  const r=await api('/api/suggest?ship='+encodeURIComponent(ship));
  el('sug').innerHTML=tbl(r,[{t:'方案',k:'方案'},{t:'预期效果',k:'效果'},{t:'代价',k:'代价'}])+
    '<div class="muted" style="margin-top:6px">最终决策由 PMC 人工确定；可到【方案模拟】量化对比</div>';
}

/* ============ 交检预测（纯工时法） ============ */
const EST_TEAMS=['初装','机管','电装','甲板','终装'];
async function renderEstimate(){
  let h='<h2>交检预测（只算剩余工时）</h2>';
  h+='<div class="panel"><div class="row">测算模型：'+
     '<select id="est_mode" onchange="loadEstimate()">'+
     '<option value="queue">标准模型（受单船人数上限约束）</option>'+
     '<option value="ideal">理想模型（不考虑区域/班组数/单船人数，只看时间）</option>'+
     '<option value="normal_jit">标准 + 准时制 JIT（受单船人数约束，卡在交检日完成）</option>'+
     '<option value="jit">理想 + 准时制 JIT（不限产能，卡在交检日完成）</option>'+
     '</select></div>'+
     '<div class="muted" style="margin-bottom:8px">'+
     '<b>标准模型</b>：一条船每天最多占用「单船人数上限 × 人均工时」（默认 5 人），'+
     '符合现场实际站位能力；<b>理想模型</b>：一条船可动用该班组的全部产能，'+
     '算出的是理论最快完成日，用于判断"如果不受站位限制，交检来不来得及"。'+
     '<br><b>准时制 JIT 系列</b>：产能口径同上，但<b>按交检日紧迫度排队</b>'+
     '（交检早的先吃产能），让有余量的船给要交的船让路 —— 与【排产与甘特图】页的 '+
     'JIT 口径一致。若结果和标准/理想模型一样，说明所有船的交检日已经很紧、没有余量可让。</div>'+
     '<div class="row">测算班组：'+
     '<label><input type="checkbox" id="est_all" checked onchange="toggleEstAll()"> 五个组同时看（综合）</label>'+
     '</div><div class="row" id="est_team_box">'+
     EST_TEAMS.map(t=>'<label><input type="checkbox" class="est_t" value="'+t+'"> '+t+'</label>').join('')+
     ' </div><div class="row">'+
     ' <button onclick="loadEstimate()">生成预测</button>'+
     ' <button class="green" onclick="exportEstimate()">⬇ 导出 Excel</button></div>'+
     ' <button class="gray" onclick="exportAudit()">⬇ 导出工时核算明细（逐道工序对账）</button></div>'+
     '<div class="muted">口径：每个班组按「剩余工时 ÷ 每天能占用的工时」算需要几天，'+
     '船按优先级排队（重点船先用产能），再按生产日历换算成日期；'+
     '一条船取各班组里最晚的那个作为预测交检。<b>不考虑工序前后置与区域约束</b>，'+
     '所以这是"理想产能上限"下的最快可能，实际通常更晚。</div>'+
     '<div id="estmsg"></div></div><div id="estbody"></div>';
  el('main').innerHTML=h;
  await loadEstimate();
}
function toggleEstAll(){
  const all=el('est_all').checked;
  document.querySelectorAll('.est_t').forEach(i=>{i.checked=!all && false; i.disabled=all;});
  if(all) loadEstimate();
}
function estTeams(){
  if(el('est_all').checked) return null;
  const v=[];
  document.querySelectorAll('.est_t').forEach(i=>{if(i.checked)v.push(i.value)});
  return v.length?v:null;
}
async function loadEstimate(){
  const teams=estTeams();
  const mode=(el('est_mode')&&el('est_mode').value)||'queue';
  let url='/api/estimate?mode='+mode;
  if(teams) url+='&'+teams.map(t=>'teams='+encodeURIComponent(t)).join('&');
  el('estbody').innerHTML='<div class="muted">测算中…</div>';
  const r=await api(url);
  const sel=r.teams||[];
  const _mn={queue:'标准模型',ideal:'理想模型',jit:'理想+准时制 JIT',normal_jit:'标准+准时制 JIT'}[mode]||mode;
  let h='<div class="panel"><h3>综合预测（'+sel.join(' / ')+'）'+
    '<span class="muted" style="font-size:13px;font-weight:400">　模型：'+_mn+'</span></h3>';
  h+='<div class="cards">';
  sel.forEach(tc=>{
    const d=r.by_team[tc]||{};
    const tt=d.temp_task_hours||0;
    h+='<div class="card"><div class="k">'+tc+' 剩余工时</div><div class="v">'+(d.total_remain||0)+
       '</div><div class="d">每天可用 '+(d.daily_hours||0)+'H，'+ (d.workers||0)+' 人'+
       (tt?('　<span class="red">临时任务 '+tt+'H</span>'):'')+'</div></div>';
  });
  const late=r.by_ship.filter(s=>s.delay_days>0).length;
  h+='<div class="card"><div class="k">预计延期船</div><div class="v" style="color:#c0392b">'+late+
     '</div><div class="d">共 '+r.by_ship.length+' 艘在制船</div></div></div>';
  let cols=[{t:'船号',k:'ship_no'},{t:'船型',k:'ship_type'},{t:'优先级',k:'priority'},
    {t:'计划交检',k:'plan_inspection'},
    {t:'预测交检',k:'predicted',f:v=>v||'—'},
    {t:'延期(生产日)',k:'delay_days',f:v=>v>0?'<span class="red">'+v+'</span>':(v<0?'<span class="red">—</span>':'<span class="green">0</span>')},
    {t:'风险',k:'risk',f:riskTag}];
  sel.forEach(tc=>{
    cols.push({t:tc+'剩余H',k:'teams',f:(v,r2)=>{const t=(v||{})[tc]||{};return t.remain||0;}});
    cols.push({t:tc+'预测',k:'teams',f:(v,r2)=>{const t=(v||{})[tc]||{};return t.finish||'—';}});
  });
  h+=tbl(r.by_ship,cols);
  h+='</div>';
  // 各班组的排队明细
  sel.forEach(tc=>{
    const d=r.by_team[tc]||{};
    h+='<div class="panel"><h3>'+tc+' 排队测算</h3>'+
       '<div class="muted" style="margin-bottom:6px">每天可用 '+(d.daily_hours||0)+
       ' H（班组总产能 '+(d.team_capacity||0)+' H / '+(d.workers||0)+' 人），按优先级排队</div>';
    h+=tbl(d.rows||[],[{t:'船号',k:'ship_no'},{t:'船型',k:'ship_type'},
      {t:'剩余工时H',k:'remain'},{t:'需要天数',k:'days'},{t:'前面排队天数',k:'wait_days'},
      {t:'预测完成',k:'finish',f:v=>v||'—'}]);
    h+='</div>';
  });
  el('estbody').innerHTML=h;
}
async function exportAudit(){
  const ship=prompt('导出哪条船的核算明细？直接留空 = 全部船','');
  if(ship===null)return;
  el('estmsg').innerHTML='<div class="msg info">正在生成对账明细…</div>';
  const r=await post('/api/audit',{act:'export',ship:ship||null});
  if(r.error){el('estmsg').innerHTML='<div class="msg err">'+r.error+'</div>';return}
  el('estmsg').innerHTML='<div class="msg ok">已导出：'+r.path+'</div>';
  alert('已导出工时核算明细：\n'+r.path+
    '\n\n逐道工序列出：班组情况表状态/工时、完成情况表状态/剩余、选配项、是否计入及原因，'+
    '可直接和你手工统计的数字对账。');
}
async function exportEstimate(){
  el('estmsg').innerHTML='<div class="msg info">正在导出…</div>';
  const mode=(el('est_mode')&&el('est_mode').value)||'queue';
  const r=await post('/api/estimate',{act:'export',teams:estTeams(),mode:mode});
  if(r.error){el('estmsg').innerHTML='<div class="msg err">'+r.error+'</div>';return}
  el('estmsg').innerHTML='<div class="msg ok">已导出：'+r.path+'</div>';
  alert('已导出：\n'+r.path);
}

/* ============ 班组周 / 月计划 ============ */
let TP_ST={ver:''};   // 选中的排产版本（空=当前正式版本）
async function renderTeamPlan(){
  let h='<h2>班组计划（周 / 月）</h2>';
  h+='<div class="panel"><div class="row">'+
     '班组 <select id="tp_team">'+['初装','机管','电装','甲板','终装'].map(t=>'<option>'+t+'</option>').join('')+'</select>'+
     ' 计划类型 <select id="tp_mode" onchange="loadTeamPlan()">'+
       '<option value="week">未来 7 天周计划</option>'+
       '<option value="month">下个月月计划</option></select>'+
     ' 起始日 <input type="date" id="tp_date" value="'+fmtDate(new Date())+'" onchange="loadTeamPlan()">'+
     ' <button onclick="loadTeamPlan()">生成</button>'+
     ' <button class="green" onclick="exportTeamPlan()">⬇ 导出该类型 Excel（五个班组各一张表）</button>'+
     '</div><div class="row">'+
     ' 计划口径 <select id="tp_ver" onchange="tpVerChange()"><option value="">当前正式版本</option></select>'+
     ' <span class="muted">标准口径含舱室上人限制；理想口径放开单船人数与同舱区班组数上限，只受船体脱模/甲板/硬顶等大节点与班组产能约束。两套计划可分别出周计划、月计划。</span>'+
     '</div><div class="muted">计划内容取自所选排产版本的结果，并附上每道工序的完成情况'+
     '（与单船实时表、班组周计划交叉核对后的结果）</div>'+
     '<div id="tpmsg"></div></div><div id="tpbody"></div>';
  el('main').innerHTML=h;
  await loadTeamPlan();
}
function tpVerChange(){TP_ST.ver=el('tp_ver').value;loadTeamPlan();}
async function tpFillVersions(list){
  const sel=el('tp_ver'); if(!sel) return;
  const cur=sel.value;
  const seen={}; const opts=['<option value="">当前正式版本</option>'];
  (list||[]).forEach(v=>{
    if(seen[v.version_no]) return; seen[v.version_no]=1;
    opts.push('<option value="'+esc(v.version_no)+'">'+esc(v.version_no)+' · '+esc(v.mode_label)+
      (v.is_active?'（当前正式）':'')+'</option>');
  });
  sel.innerHTML=opts.join('');
  sel.value=(cur&&seen[cur])?cur:(TP_ST.ver&&seen[TP_ST.ver]?TP_ST.ver:'');
  TP_ST.ver=sel.value;
}
function tpModeChange(){
  const m=el('tp_mode').value;
  el('tp_end').style.display = (m==='range')?'':'none';
  loadTeamPlan();
}
function tpPreset(n){
  el('tp_mode').value='range';
  el('tp_end').style.display='';
  const d1=new Date(); const d2=new Date(Date.now()+(n-1)*864e5);
  el('tp_date').value=fmtDate(d1); el('tp_end').value=fmtDate(d2);
  loadTeamPlan();
}
function tpModeChange(){
  const m=el('tp_mode').value;
  el('tp_end').style.display=(m==='range')?'':'none';
  loadTeamPlan();
}
function tpPreset(n){
  el('tp_mode').value='range'; el('tp_end').style.display='';
  const d1=new Date(), d2=new Date(Date.now()+(n-1)*864e5);
  el('tp_date').value=fmtDate(d1); el('tp_end').value=fmtDate(d2);
  loadTeamPlan();
}
async function loadTeamPlan(){
  const team=el('tp_team').value, mode=el('tp_mode').value, date=el('tp_date').value;
  const dend=el('tp_end')?el('tp_end').value:'';
  el('tpbody').innerHTML='<div class="muted">生成中…</div>';
  let url='/api/teamplan?team='+encodeURIComponent(team)+'&mode='+mode+'&date='+date;
  if(TP_ST.ver) url+='&version='+encodeURIComponent(TP_ST.ver);
  if(mode==='range'&&dend) url+='&start='+date+'&end='+dend;
  const p=await api(url);
  if(p.error){el('tpbody').innerHTML='<div class="msg err">'+p.error+'</div>';return}
  if(p.versions) tpFillVersions(p.versions);
  const vlabel=(p.versions||[]).find(v=>v.version_no===p.version);
  const label=(mode==='week')?'未来 7 天周计划':((mode==='month')?'下个月月计划':'自定义区间计划');
  let h='<div class="panel"><h3>'+team+' · '+label+'（'+p.start+' ~ '+p.end+'）</h3>';
  h+='<div class="muted">排产版本 '+esc(p.version)+(vlabel?' · '+esc(vlabel.mode_label):'')+'</div>';
  const _done=(p.done_hours||0), _dn=(p.rows||[]).filter(r=>r.done_before>=1).length;
  h+='<div class="cards">'+
     '<div class="card"><div class="k">计划投入工时</div><div class="v">'+p.total_hours+'</div><div class="d">H</div></div>'+
     '<div class="card"><div class="k">涉及船舶</div><div class="v">'+p.summary.length+'</div><div class="d">艘</div></div>'+
     '<div class="card"><div class="k">工序条目</div><div class="v">'+((p.rows||[]).length-_dn)+'</div></div>'+
     '<div class="card"><div class="k">已完工不计入</div><div class="v">'+_dn+'</div><div class="d">道 / '+_done+'H</div></div></div>';
  if(_dn>0){
    h+='<div class="msg info">区间内有 <b>'+_dn+'</b> 道工序已被三源交替验证判定为<b>已完工</b>'+
       '（班组情况表 / 单船工序完成情况表 / 排布计划详情，任一报完成即算完成），'+
       '共 <b>'+_done+'H</b> 不计入本计划；明细中仍列出并标注"该工序已完成"，便于与班组表对照。</div>';
  }
  if(p.rest_days>0){
    h+='<div class="msg info">该区间内有 <b>'+p.rest_days+'</b> 天是非生产日（周末 / 节假日），'+
       '排产不会安排工作；想看下个月的活，把结束日往后选即可（支持跨月）。</div>';
  }
  h+='<h3>按日投入工时</h3>';
  if(p.days&&p.days.length){
    h+=tbl(p.days,[{t:'日期',k:'date'},{t:'星期',k:'weekday'},
      {t:'是否生产',k:'is_prod',f:v=>v?'<span class="green">生产</span>':'<span class="red">休息/放假</span>'},
      {t:'投入工时H',k:'hours'}]);
  } else h+=tbl(p.by_day,[{t:'日期',k:'date'},{t:'投入工时H',k:'hours'}]);
  h+='<h3>按船汇总</h3>';
  h+=tbl(p.summary,[{t:'船号',k:'ship_no'},{t:'船型',k:'ship_type'},
    {t:'工序数',k:'tasks'},{t:'投入工时H',k:'hours'},{t:'施工天数',k:'days'},
    {t:'开始',k:'first'},{t:'结束',k:'last'},
    {t:'已完工工序',k:'done_tasks'},{t:'已完工H(不计入)',k:'done_hours'}]);
  h+='</div><div class="panel"><h3>明细（日期 × 船 × 工序）</h3>';
  h+=tbl(p.rows,[{t:'日期',k:'work_date'},{t:'星期',k:'weekday'},{t:'船号',k:'ship_no'},
    {t:'船型',k:'ship_type'},{t:'工序编码',k:'process_code'},{t:'工序名称',k:'process_name'},
    {t:'人数',k:'workers'},{t:'工时H',k:'planned_hours'},{t:'区域',k:'zone'},
    {t:'该工序已完成',k:'done_before',f:v=>v>=1?'<span class="green">是</span>':'否'}]);
  h+='</div>';
  el('tpbody').innerHTML=h;
}
async function exportTeamPlan(){
  const mode=el('tp_mode').value;
  el('tpmsg').innerHTML='<div class="msg info">正在导出…</div>';
  const body={mode:mode};
  if(TP_ST.ver) body.version=TP_ST.ver;
  const r=await post('/api/export/teamplan',body);
  if(r.error){el('tpmsg').innerHTML='<div class="msg err">'+r.error+'</div>';return}
  el('tpmsg').innerHTML='<div class="msg ok">已导出：'+r.path+'</div>';
  alert('已导出：\n'+r.path+'\n\n五个班组各一张工作表，另附总览表');
}

/* ============ 产能分析（缺人时段 / 加人模拟 / 加班组合） ============ */
async function renderAnalysis(){
  const d=await api('/api/settings');
  const ships=d.ships||[];
  let h='<h2>产能分析</h2>';
  /* --- 1. 缺口 --- */
  const g=await api('/api/gap');
  h+='<div class="panel"><h3>① 哪个时间段、哪个班组缺人</h3>';
  if(!g.gap.length){
    h+='<div class="msg ok">当前排程没有出现产能缺口（或尚未排产）</div>';
  }else{
    h+='<div class="muted" style="margin-bottom:6px">只统计"当周能干却没干成"的工时；'+
       '负荷没打满却有活排队，说明是被单船人数上限 / 区域互斥 / 前后置卡住，加人作用有限</div>';
    h+=tbl(g.gap,[{t:'周起始',k:'week_start'},{t:'周',k:'week_label'},{t:'班组',k:'team'},
      {t:'负荷率',k:'load_rate',f:v=>'<span class="'+loadColor(v)+'">'+v.toFixed(0)+'%</span>'},
      {t:'已排H',k:'planned'},{t:'可用H',k:'capacity'},
      {t:'排队工序',k:'queued_tasks'},
      {t:'未排上H',k:'gap_hours'},
      {t:'★真缺口H',k:'real_gap',f:v=>v>0?'<span class="red">'+v+'</span>':'<span class="green">0</span>'},
      {t:'建议补人',k:'need_workers',f:v=>v>0?'<span class="red">'+v+' 人</span>':'—'},
      {t:'判断',k:'reason'}]);
  h+='<div class="muted" style="margin-top:6px">'+
     '<b>未排上H</b> = 该周排产没排掉的工时（可能是缺人，也可能是被约束卡住）；'+
     '<b>★真缺口H</b> = 只统计产能已用到 85% 以上的周，也就是<b>真正靠加人能解决的部分</b>。'+
     '两者差得越多，说明越不是缺人的问题，而是前后置/区域/单船人数上限在卡。</div>';
  }
  h+='</div>';

  /* --- 2. 五组综合加人方案（新增，放在单个班组模拟之前）--- */
  h+='<div class="panel"><h3>② 五个组一起：加多少人能满足主计划</h3>'+
     '<div class="row">允许延期 <input type="number" id="hc_tol" value="0" style="width:70px"> 天'+
     ' <button onclick="loadHeadcount()">算出方案</button></div>'+
     '<div class="muted">按"累计需求 vs 累计产能"反推：把每个班组要干的活按交检日从近到远排开，'+
     '逐条累加必须在那天之前干完的工时，再看现有人力够不够，不够的部分折算成人数。'+
     '允许延期填几天，就会相应少加一些人。</div>'+
     '<div id="hc_body"></div></div>';

  /* --- 3. 加人模拟（单个班组）--- */
  h+='<div class="panel"><h3>③ 单个班组：加 1 个 / 2 个 / 3 个…人会怎样</h3><div class="row">'+
     '班组 <select id="wt_team">'+['初装','机管','电装','甲板','终装'].map(t=>'<option>'+t+'</option>').join('')+'</select>'+
     ' 关注船 <select id="wt_ship"><option value="">(全部船)</option>'+
       ships.map(s=>'<option>'+s.ship_no+'</option>').join('')+'</select>'+
     ' 人员类型 <select id="wt_type"><option value="0">熟练工</option><option value="1">新人</option></select>'+
     ' 单船人数上限放宽到 <input type="number" id="wt_max" value="0" style="width:60px" title="0=不改">'+
     ' <button onclick="runWhatIfAdd()">运行</button></div>'+
     '<div id="wt_add_body"></div></div>';

  /* --- 3. 加班 / 周日组合 --- */
  h+='<div class="panel"><h3>③ 一周加班几天 / 熟练工周日上班，能不能做出来</h3><div class="row">'+
     '关注船 <select id="wm_ship"><option value="">(全部船)</option>'+
       ships.map(s=>'<option>'+s.ship_no+'</option>').join('')+'</select>'+
     ' <button onclick="runWhatIfWorktime()">运行组合矩阵</button></div>'+
     '<div class="muted">会对「每周加班 0/2/3/4/5/6/7 天 × 周日上班 是/否」共 14 种组合各跑一次排程，约需 30 秒</div>'+
     '<div id="wm_body"></div></div>';
  el('main').innerHTML=h;
  await loadHeadcount();
}
/* ---- 五组综合人力建议 ---- */
async function loadHeadcount(){
  const box=el('hc_body');
  if(!box)return;
  box.innerHTML='<div class="muted">计算中…</div>';
  const tol=el('hc_tol')?(+el('hc_tol').value||0):0;
  const r=await api('/api/headcount?tolerance='+tol);
  if(!r.teams){box.innerHTML='<div class="muted">（暂无数据）</div>';return}
  let h='<div class="cards">';
  h+='<div class="card"><div class="k">合计需加人</div><div class="v" style="color:#c0392b">'+r.total_add+
     '</div><div class="d">人（五个组合计）</div></div>';
  r.teams.forEach(t=>{
    h+='<div class="card"><div class="k">'+t.team+'</div><div class="v">'+
       (t.add>0?('<span style="color:#c0392b">+'+t.add+'</span>'):'0')+
       '</div><div class="d">现有 '+t.workers+' 人 → 需 '+t.need_workers+' 人</div></div>';
  });
  h+='</div>';
  h+=tbl(r.teams,[{t:'班组',k:'team'},
    {t:'现有人数',k:'workers'},
    {t:'剩余工时H',k:'total_remain'},
    {t:'负责船数',k:'ships'},
    {t:'每人每天H',k:'per_day_hours'},
    {t:'峰值需人',k:'peak_need_workers',f:v=>v.toFixed(1)},
    {t:'需要人数',k:'need_workers'},
    {t:'需加人数',k:'add',f:v=>v>0?'<span class="red">+'+v+'</span>':'<span class="green">0</span>'},
    {t:'最紧的船',k:'tight_ship'},
    {t:'卡在',k:'tight_date'}]);
  const plan={};
  r.teams.forEach(t=>{if(t.add>0)plan[t.team]=t.add});
  h+='<div class="row"><button class="green" onclick=\'simHeadcount('+
     JSON.stringify(plan).replace(/'/g,"\\'")+
     ')\'>▶ 按这个方案模拟排产（五组同时加人）</button>'+
     ' <span class="muted">会跑一次正式排程，看还剩几艘船延期</span></div>';
  h+='<div id="hc_sim"></div>';
  box.innerHTML=h;
}
async function simHeadcount(plan){
  el('hc_sim').innerHTML='<div class="msg info">正在按方案模拟排产…</div>';
  const r=await post('/api/headcount',{act:'sim',plan:plan,horizon:120,freeze:2});
  if(r.error){el('hc_sim').innerHTML='<div class="msg err">'+r.error+'</div>';return}
  el('hc_sim').innerHTML='<div class="msg '+(r.delay_ships>0?'info':'ok')+'">'+
    '模拟结果：延期船 <b>'+r.delay_ships+'</b> 艘，总延期 '+r.total_delay+
    ' 天，排不完 '+r.unfinished_ships+' 艘（排程 '+r.metrics.rows+' 条）</div>'+
    '<div class="muted">方案：'+
    Object.keys(r.plan).map(k=>k+'+'+r.plan[k]).join('、')+'</div>';
}
async function runWhatIfAdd(){
  const body={team:el('wt_team').value, ship_no:el('wt_ship').value||null,
    new_workers:+el('wt_type').value, ship_max_override:+el('wt_max').value||0,
    counts:[1,2,3,5,8,10], horizon:120};
  el('wt_add_body').innerHTML='<div class="msg info">正在模拟（每种加人方案各跑一次排程）…</div>';
  const r=await post('/api/whatif/add',body);
  let h='<div class="msg '+(r.rows.some(x=>x.delay===0)?'ok':'info')+'">'+
        '基准：总延期 '+r.base_delay+' 天，排不完 '+r.base_unfinished+' 艘</div>';
  h+=tbl(r.rows,[{t:'增加人数',k:'add'},{t:'班组',k:'team'},{t:'类型',k:'worker_type'},
    {t:'关注船',k:'ship_no'},
    {t:'预测交检',k:'predicted',f:v=>v||'<span class="red">排不完</span>'},
    {t:'该船延期',k:'delay',f:v=>v<0?'<span class="red">排不完</span>':(v>0?'<span class="red">'+v+' 天</span>':'<span class="green">0</span>')},
    {t:'全厂延期船',k:'delay_ships'},{t:'总延期',k:'total_delay'},{t:'排不完',k:'unfinished_ships'}]);
  el('wt_add_body').innerHTML=h;
}
async function runWhatIfWorktime(){
  el('wm_body').innerHTML='<div class="msg info">正在跑 14 种组合，约 30 秒…</div>';
  const r=await post('/api/whatif/worktime',{ship_no:el('wm_ship').value||null,horizon:120});
  let h=tbl(r.rows,[{t:'每周加班',k:'ot_days'},{t:'周日上班',k:'work_sun',f:v=>v?'是':'否'},
    {t:'组合',k:'desc'},{t:'延期船数',k:'delay_ships'},{t:'总延期',k:'total_delay'},
    {t:'排不完',k:'unfinished_ships'},
    {t:'该船预测',k:'ship_predicted',f:v=>v||'—'},
    {t:'该船延期',k:'ship_delay',f:v=>v===undefined||v===null?'—':(v<0?'<span class="red">排不完</span>':(v>0?'<span class="red">'+v+'</span>':'<span class="green">0</span>'))}]);
  el('wm_body').innerHTML=h;
}

/* ============ 方案模拟 ============ */
async function renderScenario(){
  const d=await api('/api/scenario_meta');
  let h='<h2>方案模拟</h2><div class="panel"><h3>① 选择方案</h3><div class="row">';
  d.presets.forEach(p=>{
    h+='<label><input type="checkbox" class="sc" value="'+p.id+'"'+(p.def?' checked':'')+'> '+p.name+'</label>';});
  h+='</div><div class="row">起始日 <input type="date" id="ssd" value="'+d.today+'"> 展望期 '+
     '<input type="number" id="shz" value="120" style="width:80px"> '+
     '<button onclick="doScenario()">▶ 运行对比</button></div>'+
     '<div class="muted">每个方案都会完整重跑一次排程，方案越多耗时越长（单个约 2 秒）</div>'+
     '<div id="scmsg"></div></div><div id="scbody"></div>';
  el('main').innerHTML=h;
}
async function doScenario(){
  const ids=[...document.querySelectorAll('.sc:checked')].map(x=>x.value);
  if(!ids.length){alert('请至少选择一个方案');return}
  el('scmsg').innerHTML='<div class="msg info">正在运行…</div>';
  const r=await post('/api/scenario',{ids:ids,start:el('ssd').value,horizon:+el('shz').value});
  let h='<div class="panel"><h3>② 对比结果</h3>';
  h+=tbl(r.rows,[{t:'方案',k:'name'},{t:'目标值',k:'score',f:v=>v===null?'-':v.toFixed(0)},
    {t:'延期船',k:'delay_ships'},{t:'总延期',k:'delay_days'},{t:'排不完船',k:'unfinished'},
    {t:'未完工序',k:'unfinished_tasks'},{t:'换船次数',k:'switches'},
    {t:'峰值负荷',k:'peak',f:v=>v.toFixed(0)+'%'},{t:'排程行数',k:'rows'}]);
  if(r.best)h+='<div class="msg ok">推荐方案：<b>'+r.best.name+'</b>（延期船 '+r.best.delay_ships+
     ' 艘，排不完 '+r.best.unfinished+' 艘，总延期 '+r.best.delay_days+' 生产日）</div>';
  h+='<div class="muted">评分口径：排不完的船最严重（按 60 生产日折算），其次是延期船数与延期天数</div></div>';
  el('scbody').innerHTML=h;
  el('scmsg').innerHTML='';
  S.version=r.version;loadStatus();
}

/* ============ 设置 ============ */
async function renderSettings(){
  const d=await api('/api/settings');
  const ships=d.ships||[];
  const _sm=SHIP_MSG; SHIP_MSG='';   // 批量操作提示只显示一次
  let h='<h2>系统设置</h2><div class="panel"><h3>① 班组人力</h3>'+
    '<div class="muted" style="margin-bottom:8px">当天产能 = 熟练工×熟练工日工时 + 新人×新人日工时。'+
    '例：终装 12×11 + 4×4 = 148H</div>';
  h+='<div class="tbl-wrap"><table><thead><tr><th>班组</th><th>熟练工</th><th>新人</th>'+
     '<th>熟练工日H</th><th>新人日H</th><th>周六</th><th>周日</th><th>加班H</th>'+
     '<th>单船上限</th><th>赶工上限</th><th>日产能H</th></tr></thead><tbody>';
  d.teams.forEach(t=>{
    h+='<tr><td>'+t.team_code+'</td>'+
      '<td><input style="width:60px" data-t="'+t.team_code+'" data-f="skilled" value="'+t.skilled+'"></td>'+
      '<td><input style="width:60px" data-t="'+t.team_code+'" data-f="new_workers" value="'+t.new_workers+'"></td>'+
      '<td><input style="width:60px" data-t="'+t.team_code+'" data-f="sh" value="'+t.sh_per_day+'"></td>'+
      '<td><input style="width:60px" data-t="'+t.team_code+'" data-f="nh" value="'+t.nh_per_day+'"></td>'+
      '<td><input type="checkbox" data-t="'+t.team_code+'" data-f="work_sat"'+(t.work_sat?' checked':'')+'></td>'+
      '<td><input type="checkbox" data-t="'+t.team_code+'" data-f="work_sun"'+(t.work_sun?' checked':'')+'></td>'+
      '<td><input style="width:60px" data-t="'+t.team_code+'" data-f="overtime" value="'+t.overtime+'"></td>'+
      '<td><input style="width:60px" data-t="'+t.team_code+'" data-f="ship_max" value="'+t.ship_max+'"></td>'+
      '<td><input style="width:60px" data-t="'+t.team_code+'" data-f="ship_max_u" value="'+t.ship_max_u+'"></td>'+
      '<td><b>'+(t.skilled*t.sh_per_day+t.new_workers*t.nh_per_day).toFixed(0)+'</b></td></tr>';
  });
  h+='</tbody></table></div><div class="row" style="margin-top:10px">'+
     '<button onclick="saveTeams()">💾 保存班组配置</button></div><div id="tmsg"></div></div>';
  h+='<div class="panel"><h3>② 船只优先级与锁定</h3>';

  /* ---- 批量筛选：几十条船一键剔除 / 放回 ---- */
  const nIn=(d.ships||[]).filter(s=>s.in_plan).length;
  const nEx=(d.ships||[]).filter(s=>!s.in_plan).length;
  h+='<div style="background:#f7f9fc;border:1px solid #e3e8ef;border-radius:8px;padding:10px;margin-bottom:10px">'+
    '<div class="row" style="align-items:center;gap:8px;flex-wrap:wrap">'+
      '<b>批量筛选</b>'+
      '<span class="muted">已勾选 <b id="shipsel">0</b> 艘 ｜ 参与排产 '+
      '<b class="t-grn" style="padding:0 4px">'+nIn+'</b> 艘 ｜ 已剔除 '+
      '<b class="t-red" style="padding:0 4px">'+nEx+'</b> 艘</span>'+
      '<button class="gray" onclick="shipChkAll(1)">全选</button>'+
      '<button class="gray" onclick="shipChkAll(0)">全不选</button>'+
      '<button class="gray" onclick="shipChkInvert()">反选</button>'+
      '<button class="red" onclick="shipBulk(\'exclude\')">剔除勾选的船</button>'+
      '<button onclick="shipBulk(\'include\')">放回勾选的船</button>'+
    '</div>'+
    '<div class="row" style="margin-top:8px;align-items:flex-start;gap:10px;flex-wrap:wrap">'+
      '<textarea id="ship_paste" style="width:360px;height:62px;font-family:inherit" '+
      'placeholder="把不要的船号粘贴到这里（逗号 / 空格 / 换行分隔都行）&#10;例如：A32S104 A32S105, A70S02"></textarea>'+
      '<div style="display:flex;flex-direction:column;gap:6px">'+
        '<button class="red" onclick="shipPaste(\'exclude\')">剔除这些船</button>'+
        '<button onclick="shipPaste(\'keep_only\')">只保留这些船（其余全剔除）</button>'+
      '</div>'+
      '<div style="display:flex;flex-direction:column;gap:6px">'+
        '<button class="red" onclick="shipBulkAct(\'exclude_all\')">全部剔除</button>'+
        '<button onclick="shipBulkAct(\'include_all\')">全部放回</button>'+
      '</div>'+
    '</div>'+
    '<div class="muted" style="margin-top:6px">「剔除」= 该船暂不参与排产（置为<b>暂缓</b>），随时可放回，不会删数据。'+
    '最快用法：把主计划要排的船号粘进文本框，点【只保留这些船】。改完到【排产与甘特图】重新排产即可。</div>'+
    (_sm?'<div class="msg ok" style="margin-top:6px">'+_sm+'</div>':'')+
    '<div id="shipmsg"></div>'+
  '</div>';

  /* ---- 复制船：同型号换个序号就是一条新船，工时按新船型刷新 ---- */
  CP_SHIPS=(d.ships||[]).map(s=>({ship_no:s.ship_no,ship_type:s.ship_type||'',in_plan:!!s.in_plan}));
  h+='<div style="background:#f0f7f4;border:1px solid #cfe5da;border-radius:8px;padding:10px;margin-bottom:10px">'+
    '<div class="row" style="align-items:center;gap:8px;flex-wrap:wrap">'+
      '<b>复制船</b>'+
      '<span class="muted">从已有船复制出一条新船（换个序号），工时自动刷新</span>'+
    '</div>'+
    '<div class="row" style="margin-top:8px;align-items:center;gap:8px;flex-wrap:wrap">'+
      '<span>源船</span><select id="cp_src" onchange="cpOnSrc()" style="min-width:150px">'+
        '<option value="">（选择要复制的船）</option>'+
        (d.ships||[]).map(s=>'<option value="'+s.ship_no+'">'+s.ship_no+
          (s.ship_type?('　'+s.ship_type):'')+(s.in_plan?'':'（已剔除）')+'</option>').join('')+
      '</select>'+
      '<span>新船号</span><input id="cp_no" style="width:130px" placeholder="如 A45S009">'+
      '<span>船型</span><input id="cp_type" style="width:120px" placeholder="留空=沿用源船">'+
      '<button class="gray" onclick="cpFillNo()">按源船型取下一个序号</button>'+
    '</div>'+
    '<div class="row" style="margin-top:8px;align-items:center;gap:14px;flex-wrap:wrap">'+
      '<span>工时来源</span>'+
      '<label style="display:inline-flex;align-items:center;gap:4px"><input type="radio" '+
        'name="cp_mode" value="template" checked> 按新船型模板刷新（推荐）</label>'+
      '<label style="display:inline-flex;align-items:center;gap:4px"><input type="radio" '+
        'name="cp_mode" value="clone"> 照搬源船工序，完成量清零</label>'+
      '<label style="display:inline-flex;align-items:center;gap:4px"><input type="checkbox" '+
        'id="cp_dates"> 沿用源船时间节点（默认留空自己填）</label>'+
    '</div>'+
    '<div class="row" style="margin-top:8px;gap:8px;align-items:center">'+
      '<button class="gray" onclick="cpPreview()">预览会复制什么</button>'+
      '<button onclick="cpDo()">📋 执行复制</button>'+
    '</div>'+
    '<div id="cpmsg"></div><div id="cpview"></div>'+
    '<div class="muted" style="margin-top:6px">新船会以<b>在制</b>状态进入总计划。'+
    '若时间节点留空，排产时会被船底启动门禁拦住，补填后即可排产。'+
    '历史报工（完成情况表）<b>不会</b>复制过来，新船是干净的。</div>'+
  '</div>';

  /* ---- 船底启动门禁：船底未启动（没填 / 未到）的船不排产 ---- */
  let kg={keel_gate:(d.keel_gate===undefined?1:d.keel_gate),
          keel_rule:(d.keel_rule||'auto'), blocked:[], blocked_count:0, base_day:''};
  try{ kg=await api('/api/keel_gate'); }catch(e){}
  KEEL_BLOCKED=kg.blocked||[];
  h+='<div style="background:#fff8e6;border:1px solid #f0d9a0;border-radius:8px;padding:10px;margin-bottom:10px">'+
    '<div class="row" style="align-items:center;gap:10px;flex-wrap:wrap">'+
      '<b>船底启动门禁</b>'+
      '<label style="display:inline-flex;align-items:center;gap:4px">'+
        '<input type="checkbox" id="keel_gate"'+(kg.keel_gate?' checked':'')+
        ' onchange="setKeelGate()"> 船底未启动的船不参与排产</label>'+
      '<span>判定口径</span><select id="keel_rule" onchange="setKeelGate()">'+
        '<option value="auto"'+(kg.keel_rule==='auto'?' selected':'')+'>实际优先，计划已到也算（默认）</option>'+
        '<option value="plan"'+(kg.keel_rule==='plan'?' selected':'')+'>只看计划船底启动日</option>'+
        '<option value="sale"'+(kg.keel_rule==='sale'?' selected':'')+'>只看实际船底启动日</option>'+
      '</select>'+
    '</div>'+
    '<div class="muted" style="margin-top:6px">主计划里"船底启动"没填、或日期还没到的船，视为<b>船底未启动</b>：'+
    '整船不进排产，也就不会出现在班组周计划里。改完到【排产与甘特图】重新排产生效。</div>'+
    (kg.blocked_count
      ? '<details style="margin-top:6px"><summary>按当前口径会被排除 <b class="t-red" style="padding:0 4px">'+
        kg.blocked_count+'</b> 艘，点开看明细</summary>'+
        '<div class="tbl-wrap" style="margin-top:6px"><table><thead><tr>'+
        '<th>船号</th><th>船型</th><th>计划船底启动</th><th>实际船底启动</th>'+
        '<th>计划交检</th><th>剩余H</th><th>原因</th></tr></thead><tbody>'+
        KEEL_BLOCKED.map(function(b){
          return '<tr><td>'+b.ship_no+'</td><td>'+(b.ship_type||'')+'</td>'+
            '<td>'+(b.plan_start?fmtDate(b.plan_start):'<span class="red">未填</span>')+'</td>'+
            '<td>'+(b.sale_start?fmtDate(b.sale_start):'<span class="red">未填</span>')+'</td>'+
            '<td>'+(b.plan_inspection?fmtDate(b.plan_inspection):'-')+'</td>'+
            '<td>'+Math.round(b.remaining_hours||0)+'</td>'+
            '<td>'+b.reason+'</td></tr>';
        }).join('')+
        '</tbody></table></div></details>'
      : '<div class="muted" style="margin-top:6px">当前没有因船底未启动被排除的船</div>')+
    '<div id="keelmsg"></div>'+
  '</div>';

  /* ---- 船只表格（带勾选框，整页重绘后仍保持勾选）---- */
  h+='<div class="tbl-wrap"><table><thead><tr>'+
    '<th style="width:34px"><input type="checkbox" class="shipchk-all" '+
      'onchange="shipChkAll(this.checked?1:0)" title="全选 / 全不选"></th>'+
    '<th>船号</th><th>参与排产</th><th>计划船底启动（可手填）</th><th>船型</th>'+
    '<th>计划交检（可直接改）</th>'+
    '<th>人工优先级</th><th>必须按期</th><th>状态</th><th>固定人数</th>'+
    '<th>其中新人</th><th>剩余H</th><th style="width:112px">操作</th></tr></thead><tbody>';
  (d.ships||[]).forEach(function(r){
    const on=!!r.in_plan;
    const _rem=Number(r.remaining_hours||0);
    h+='<tr'+(on?'':' style="background:#fafafa;opacity:.7"')+'>'+
      '<td><input type="checkbox" class="shipchk" data-s="'+esc(r.ship_no)+'"'+
        (on?' checked':'')+' onchange="shipSelCnt();shipKeep()"></td>'+
      '<td>'+esc(r.ship_no)+(r.overdue?' <span class="red">[已逾期]</span>':'')+
        (/\s/.test(r.ship_no||'')?' <span class="t-yel" title="船号里带空格，建议用操作里的「去空格」修正">[含空格]</span>':'')+
      '</td>'+
      '<td>'+({在制:'<span class="tag t-grn">参与</span>',暂缓:'<span class="tag t-yel">暂缓</span>',
        冻结:'<span class="tag t-yel">冻结</span>',未开工:'<span class="tag t-gry">未开工</span>',
        完工:'<span class="tag t-gry">已完工</span>'}[r.status]||(on?'<span class="tag t-grn">参与</span>':'<span class="tag t-gry">已剔除</span>'))+'</td>'+
      '<td><input type="date" style="width:132px" data-s="'+esc(r.ship_no)+'" class="shipplan"'+
        ' value="'+String(r.plan_start||'').slice(0,10)+'"'+
        ' title="填了才参与排产；留空=被船底启动门禁拦住"></td>'+
      '<td>'+esc(r.ship_type||'')+'</td>'+
      '<td><input type="date" style="width:132px" data-s="'+esc(r.ship_no)+'" class="shippic" value="'+
        String(r.plan_inspection||'').slice(0,10)+'"></td>'+
      '<td><select data-s="'+esc(r.ship_no)+'" class="shipprio">'+
        ['紧急','高','正常','低','暂缓','冻结'].map(o=>'<option'+(o===r.manual_priority?' selected':'')+'>'+o+'</option>').join('')+'</select></td>'+
      '<td><input type="checkbox" data-s="'+esc(r.ship_no)+'" class="shipmust"'+(r.must_finish_on_time?' checked':'')+'></td>'+
      '<td><select data-s="'+esc(r.ship_no)+'" class="shipst">'+
        ['在制','完工','暂缓','冻结'].map(o=>'<option'+(o===r.status?' selected':'')+'>'+o+'</option>').join('')+'</select></td>'+
      '<td><input type="number" style="width:56px" data-s="'+esc(r.ship_no)+'" class="shipfix" value="'+(r.fixed_workers||0)+'"></td>'+
      '<td><input type="number" style="width:56px" data-s="'+esc(r.ship_no)+'" class="shipfixnew" value="'+(r.fixed_new||0)+'"></td>'+
      '<td title="'+(r.remaining_stale
          ? ('统计字段里是 '+(r.remaining_hours_stat||0)+'H，这里按工序实时算出 '+_rem.toFixed(0)+'H')
          : ('共 '+((r.total_hours!==undefined)?r.total_hours:0)+'H，剩余 '+_rem.toFixed(0)+'H'))+'">'+
        '<b'+(r.remaining_stale?' class="t-yel"':'')+'>'+_rem.toFixed(0)+'</b>'+
        (r.remaining_stale?' <span class="muted" style="font-size:11px">实时</span>':'')+
        (r.task_count!==undefined?('<div class="muted" style="font-size:11px">'+r.task_count+' 道</div>'):'')+
      '</td>'+
      '<td style="white-space:nowrap">'+
        '<button class="gray" style="padding:2px 6px" title="复制这条船" '+
          'onclick="cpPick(\''+esc(r.ship_no)+'\')">复制</button> '+
        '<button class="red" style="padding:2px 6px" title="删除这条船" '+
          'onclick="shipDel(\''+esc(r.ship_no)+'\')">删除</button>'+
      '</td>'+
    '</tr>';
  });
  h+='</tbody></table></div>';
  h+='<div class="row" style="margin-top:6px;gap:10px;align-items:center;flex-wrap:wrap">'+
    '<span class="muted">共 '+(d.ships||[]).length+' 艘（含已剔除）</span>'+
    '<button class="red" style="padding:3px 10px" onclick="shipDelSel()">'+
      '🗑 删除勾选的船</button>'+
    '<button class="gray" style="padding:3px 10px" onclick="shipFixDup()">'+
      '去空格修正脏船号</button>'+
    '<span class="muted" style="font-size:12px">删除是<b>真删</b>（可从删除前的自动备份恢复）；'+
      '「去空格」把船号里的空格/制表符去掉并合并同号</span></div>';
  /* 过期船开关 */
  let ex={skip_expired:1,expired:[]};
  try{ ex=await api('/api/expired_ships'); }catch(e){}
  h+='<div class="row" style="margin-top:8px"><label>'+
     '<input type="checkbox" id="skip_exp"'+(ex.skip_expired?' checked':'')+' onchange="setSkipExpired()"> '+
     '计划交检日期早于今天的船，不进入船只设置、也不参与排产</label></div>'+
     '<div class="muted">默认开启：主计划里那些早就过了交检日的历史船会被标记为【已过期】并跳过。'+
     '如果确实有延期未交的船要排，取消勾选即可。'+
     (ex.expired&&ex.expired.length
       ? ' 当前已跳过 <b>'+ex.expired.length+'</b> 艘：'+
         ex.expired.slice(0,8).map(s=>s.ship_no+'('+String(s.plan_inspection||'').slice(0,10)+')').join('、')+
         (ex.expired.length>8?' …':'')+
         ' <button class="gray" onclick="restoreExpired()">恢复这些船</button>'
       : ' 当前没有被跳过的船')+
     '</div>';
  /* ---- 临时任务（某组在某段时间临时增加的工时）---- */
  let tt={rows:[]};
  try{ tt=await api('/api/temp_tasks'); }catch(e){}
  h+='<div class="panel"><h3>临时任务（额外工作，会占用班组产能）</h3>';
  h+='<div class="row">班组 <select id="tt_team">'+
     EST_TEAMS.map(t=>'<option>'+t+'</option>').join('')+'</select>'+
     ' 开始 <input type="date" id="tt_s"> 结束 <input type="date" id="tt_e">'+
     ' 工时 <input type="number" id="tt_h" value="40" style="width:80px"> H'+
     ' 说明 <input id="tt_n" placeholder="如：临时支援A70船" style="width:180px">'+
     ' <button onclick="addTempTask()">+ 添加</button></div>';
  h+='<div class="muted">起止日期都可以留空：都不填=算在今天；只填开始=从那天起往后 30 天摊；'+
     '只填结束=从今天摊到那天；都填=这段时间内按生产日均摊。可添加多条。</div>';
  if(tt.rows&&tt.rows.length){
    h+=tbl(tt.rows,[{t:'班组',k:'team'},
      {t:'开始',k:'start_date',f:v=>(v?String(v).slice(0,10):'（不填）')},
      {t:'结束',k:'end_date',f:v=>(v?String(v).slice(0,10):'（不填）')},
      {t:'工时H',k:'hours'},{t:'说明',k:'note'},
      {t:'启用',k:'enabled',f:(v,r)=>'<input type="checkbox"'+(v?' checked':'')+
        ' onchange="toggleTempTask('+r.id+',this.checked)">'},
      {t:'操作',k:'id',f:v=>'<button class="red" onclick="delTempTask('+v+')">删除</button>'}]);
  } else h+='<div class="muted">（暂无临时任务）</div>';
  h+='</div>';

  /* 工时统计规则 */
  let hr={team_scope_only:1};
  try{ hr=await api('/api/hour_rules'); }catch(e){}
  h+='<div class="row" style="margin-top:8px"><label>'+
     '<input type="checkbox" id="ts_only"'+(hr.team_scope_only?' checked':'')+
     ' onchange="setHourRules()"> '+
     '<b>班组情况表未列的工序，即使完成情况表里有，也不算时间</b></label></div>'+
     '<div class="muted">默认开启：以班组情况表（初装一组_总装一线 那类）为准，'+
     '它没列的活不纳入产能测算。取消勾选则完成情况表独有的工序也会计入。</div>';
  h+='<div class="row" style="margin-top:10px"><button onclick="saveShips()">💾 保存船只设置（含固定人数）</button></div>'+
     '<div class="muted">固定人数=0 表示不锁定，按班组默认的单船人数上限排产；'+
     '填了数字后，这条船每天最多就投入这么多人（可指定其中几名是新人）</div>'+
     '<div id="smsg"></div></div>';

  /* ④ 作息与加班 */
  const wt=d.worktime||{};
  h+='<div class="panel"><h3>④ 作息与加班</h3><div class="row">'+
     '每周上班：周六 <input type="checkbox" id="wt_sat"'+(wt.work_sat?' checked':'')+'>'+
     ' 周日 <input type="checkbox" id="wt_sun"'+(wt.work_sun?' checked':'')+'>'+
     ' ｜ 每周加班 <input type="number" id="wt_days" style="width:60px" value="'+(wt.ot_days_per_week||0)+'"> 天'+
     ' ｜ 每次加班 <input type="number" id="wt_hours" style="width:60px" value="'+(wt.ot_hours||0)+'"> 小时'+
     ' ｜ 新人加班 <input type="checkbox" id="wt_newot"'+(wt.new_overtime?' checked':'')+'>'+
     ' <button onclick="saveWorktime()">💾 保存并重算日历</button></div>'+
     '<div class="muted">加班会分配到每周的前 N 个工作日；新人默认不加班。'+
     '改完请到【排产与甘特图】重新排产才会生效</div>'+
     '<div id="wtmsg"></div></div>';

  /* ⑤ 节假日 */
  h+='<div class="panel"><h3>⑤ 节假日 / 停工日</h3><div class="row">'+
     '日期 <input type="date" id="hol_date"> '+
     '<select id="hol_type"><option>节假日</option><option>停工</option></select>'+
     ' <button onclick="addHoliday()">添加</button>'+
     ' <button class="gray" onclick="delHoliday()">删除该日期</button></div>';
  const hols=d.holidays||[];
  h+=tbl(hols,[{t:'日期',k:'cal_date'},{t:'类型',k:'day_type'}]);
  h+='<div class="muted" style="margin-top:6px">这些日期产能按 0 计算，且不会被自动日历覆盖</div>'+
     '<div id="holmsg"></div></div>';

  /* ⑥ 人力占用（不计算工时） */
  h+='<div class="panel"><h3>⑥ 人力占用：某条船不算工时，只占人</h3>'+
     '<div class="muted" style="margin-bottom:8px">'+
     '适合"这条船的人被挪去别处干活 / 只是占位占用"的情况：勾上后，'+
     '该船的工序不再参与排产，而是按下面的<b>时间段</b>每天直接从对应班组扣掉人数。'+
     '可以按阶段设置不同人数（前期多、后期少）。</div>'+
     '<div class="row">船号 <select id="mp_ship" onchange="loadManpower()">'+
       ships.map(s=>'<option>'+s.ship_no+'</option>').join('')+'</select>'+
     ' <label><input type="checkbox" id="mp_mode"> 按人力占用排产（不计算工序工时）</label>'+
     ' <button class="gray" onclick="addManpowerSeg()">+ 添加时间段</button></div>'+
     '<div id="mp_rows"></div>'+
     '<div class="row"><button onclick="saveManpower()">💾 保存人力占用</button></div>'+
     '<div class="muted">每格填"人数 / 其中新人"，例如 3/1 表示 3 人里 1 名新人；'+
     '日期留空表示不限</div>'+
     '<div id="mpmsg"></div></div>';
  MP_SEGS=[];
  renderManpowerSegs();
  loadManpower();
  h+='<div class="panel"><h3>③ 作业区域容量</h3>'+
     '<div class="muted" style="margin-bottom:6px">同一天同一区域允许进入的班组数（船舶空间有限，这是硬约束）</div>';
  h+=tbl(d.zones,[{t:'区域',k:'zone_code'},
    {t:'最大班组数',k:'max_teams',f:function(v,r){
      return '<input style="width:60px" data-z="'+r.zone_code+'" class="zcap" value="'+v+'">';}}]);
  h+='<div class="row" style="margin-top:10px"><button onclick="saveZones()">💾 保存区域容量</button></div></div>';
  h+='<div class="panel"><h3>⑦ 已作废的船（叉掉的船，可一键找回）</h3>'+
     '<div class="muted" style="margin-bottom:6px">在这里被叉掉的船<b>不会再被一键导入复活</b>。'+
     '点「恢复」会把它连同工序、报工记录一起从历史备份里捞回来 —— 不会丢工时。</div>'+
     '<div id="excBody"></div></div>';
  el('main').innerHTML=h;
  loadExcluded();
  shipSelCnt();
  // 整页重绘后把「复制船」面板的选择/输入恢复回来，免得用户填了一半被刷掉
  if(CP_ST.src){
    if(el('cp_src')&&[...el('cp_src').options].some(o=>o.value===CP_ST.src))
      el('cp_src').value=CP_ST.src;
    if(el('cp_no'))el('cp_no').value=CP_ST.no||'';
    if(el('cp_type'))el('cp_type').value=CP_ST.type||'';
    const rd=document.querySelector('input[name="cp_mode"][value="'+(CP_ST.mode||'template')+'"]');
    if(rd)rd.checked=true;
    if(el('cp_dates'))el('cp_dates').checked=!!CP_ST.dates;
  }
}
/* ===== 已作废船：列出 + 一键恢复 ===== */
const EXC_SEL={};
async function loadExcluded(){
  let d;
  try{ d=await post('/api/ships/excluded',{act:'list'}); }
  catch(e){ el('excBody').innerHTML='<div class="muted">读取失败：'+e.message+'</div>'; return; }
  const rows=d.rows||[];
  if(!rows.length){ el('excBody').innerHTML='<div class="muted">没有被作废的船</div>'; return; }
  let h=tbl(rows,[{t:'',k:'_c',f:(_,r)=>'<input type="checkbox" '+(EXC_SEL[r.ship_no]?'checked':'')+
      ' onchange="EXC_SEL[\''+r.ship_no+'\']=this.checked;excRefresh()">'},
    {t:'船号',k:'ship_no'},{t:'作废原因',k:'reason'},{t:'作废时间',k:'created_at'}],true);
  h+='<div class="row" style="margin-top:8px">'+
     '<span id="excSel">（未选）</span> '+
     '<button class="green" onclick="excRestore()">↩︎ 恢复选中的船</button></div>'+
     '<div class="muted" style="margin-top:6px">可用历史备份：'+
     ((d.backups&&d.backups.length)?d.backups.join('、'):'（暂无）')+'</div>';
  el('excBody').innerHTML=h;
  excRefresh();
}
function excRefresh(){
  const s=Object.keys(EXC_SEL).filter(k=>EXC_SEL[k]);
  const n=el('excSel');
  if(n)n.innerHTML=s.length?('<b>'+s.join('、')+'</b>（'+s.length+' 艘）'):'（未选）';
}
async function excRestore(){
  const s=Object.keys(EXC_SEL).filter(k=>EXC_SEL[k]);
  if(!s.length){alert('请先勾选要恢复的船');return;}
  if(!confirm('恢复这 '+s.length+' 艘船？\n'+s.join('、')+
     '\n\n将把它们连同工序与报工记录一起从历史备份里捞回来。\n恢复后请到【排产与甘特图】重新排产。'))return;
  const r=await post('/api/ships/excluded',{act:'restore',ships:s});
  Object.keys(EXC_SEL).forEach(k=>EXC_SEL[k]=false);
  await loadExcluded();
  let msg='恢复完成：\n';
  (r.restored||[]).forEach(x=>{msg+='  '+x.ship_no+'：'+(x.ok?'✓ ':'✗ ')+x.detail+'\n';});
  if(r.error) msg+='\n有失败项：'+r.error;
  alert(msg);
}
async function saveTeams(){
  const teams={};
  document.querySelectorAll('[data-t]').forEach(i=>{
    const t=i.dataset.t,f=i.dataset.f;
    if(!teams[t])teams[t]={};
    teams[t][f]=(i.type==='checkbox')?(i.checked?1:0):(isNaN(+i.value)?i.value:+i.value);
  });
  await post('/api/settings/teams',{teams:teams});
  el('tmsg').innerHTML='<div class="msg ok">已保存，下次排产生效</div>';
  setTimeout(renderSettings,600);
}
async function saveShips(){
  const ships={};
  document.querySelectorAll('.shipprio').forEach(i=>{ships[i.dataset.s]={p:i.value}});
  document.querySelectorAll('.shipmust').forEach(i=>{if(ships[i.dataset.s])ships[i.dataset.s].m=i.checked?1:0});
  document.querySelectorAll('.shipst').forEach(i=>{if(ships[i.dataset.s])ships[i.dataset.s].s=i.value});
  document.querySelectorAll('.shipfix').forEach(i=>{if(ships[i.dataset.s])ships[i.dataset.s].fw=+i.value||0});
  document.querySelectorAll('.shipfixnew').forEach(i=>{if(ships[i.dataset.s])ships[i.dataset.s].fn=+i.value||0});
  document.querySelectorAll('.shippic').forEach(i=>{if(ships[i.dataset.s])ships[i.dataset.s].pi=i.value||null});
  // 计划船底启动日：显式传 '' 表示"清空那一天"（后端据此区分"没改"和"要清空"）
  document.querySelectorAll('.shipplan').forEach(i=>{if(ships[i.dataset.s])ships[i.dataset.s].ps=i.value||''});
  await post('/api/settings/ships',{ships:ships});
  el('smsg').innerHTML='<div class="msg ok">已保存（船底启动 / 交检时间 / 优先级 / 固定人数）。'+
    '手填的日期会一直优先，再导入也不会被冲掉。改完到【排产与甘特图】重新排产生效</div>';
  setTimeout(renderSettings,500);
}
/* ============ 删除船 ============ */
async function shipDel(ship_no){
  if(!ship_no)return;
  let pv={};
  try{ pv=await post('/api/ship/delete',{act:'preview',ships:[ship_no]}); }catch(e){}
  const it=(pv.items||[])[0]||{};
  const cnt=it.child_counts||{};
  const detail=Object.keys(cnt).length
    ? Object.keys(cnt).map(function(k){return '　- '+k+'：'+cnt[k]+' 条'}).join('\n')
    : '　（没有从属数据）';
  const withTasks=confirm(
    '要删除的船：'+ship_no+'\n'+(it.ship_type?('船型：'+it.ship_type+'\n'):'')+
    (it.status?('状态：'+it.status+'\n'):'')+
    '\n会被一起清掉的数据：\n'+detail+
    (it.hours?('\n计划总工时：'+it.hours+' H'):'')+
    '\n\n点【确定】= 连工序/报工一起删干净（推荐）\n点【取消】= 只删这条船、保留工序');
  const with_tasks=withTasks?1:0;
  if(!withTasks){
    if(!confirm('再确认一次：只删船只那一行，它的 '+((cnt.tasks)||0)+
      ' 道工序会留在库里（可能变成无主的孤儿数据）。\n\n确定要这样删吗？'))return;
  }
  el('smsg').innerHTML='<div class="msg info">正在删除 '+esc(ship_no)+' …</div>';
  const r=await post('/api/ship/delete',{act:'delete',ships:[ship_no],with_tasks:with_tasks});
  if(r.errors&&r.errors.length){
    el('smsg').innerHTML='<div class="msg err">删除失败：'+esc(r.errors.join('；'))+'</div>';
    return;
  }
  SHIP_MSG='已删除 '+ship_no+(with_tasks?'（连工序一起）':'（只删船，工序保留）');
  renderSettings();
}
async function shipDelSel(){
  const sel=[...document.querySelectorAll('.shipchk')].filter(i=>i.checked)
    .map(i=>i.dataset.s).filter(Boolean);
  if(!sel.length){alert('请先在左列勾选要删除的船');return}
  if(sel.length>10&&!confirm('一次要删 '+sel.length+' 艘，确认？'))return;
  if(!confirm('确认删除这 '+sel.length+' 艘船？\n\n'+sel.join('、')+
    '\n\n将连它们的工序/报工一起清掉。此操作不可撤销（但可以从自动备份恢复）。'))return;
  const r=await post('/api/ship/delete',{act:'delete',ships:sel,with_tasks:1});
  SHIP_MSG='已删除 '+(r.deleted||0)+' 艘船'+
    (r.errors&&r.errors.length?('；失败 '+(r.errors||[]).join('；')):'');
  renderSettings();
}
async function shipFixDup(){
  const dirty=[...(CP_SHIPS||[])].map(s=>s.ship_no).filter(n=>/\s/.test(n||''));
  if(!dirty.length){alert('库里没有带空格的船号，不用修正');return}
  if(!confirm('发现 '+dirty.length+' 个带空格/制表符的船号：\n\n'+dirty.join('\n')+
    '\n\n要把空格去掉并合并到同号船吗？（工序和报工都会跟着搬过去）'))return;
  const r=await post('/api/ship/delete',{act:'fix_dup',ships:dirty});
  SHIP_MSG='船号去空格修正：成功 '+(r.fixed||[]).length+' 个'+
    ((r.fixed&&r.fixed.length)?('（'+r.fixed.join('、')+'）'):'')+
    ((r.failed&&r.failed.length)?('；跳过 '+(r.failed||[]).join('、')):'');
  renderSettings();
}
async function addTempTask(){
  const body={act:'add',team:el('tt_team').value,start:el('tt_s').value||null,
    end:el('tt_e').value||null,hours:+el('tt_h').value||0,note:el('tt_n').value||''};
  if(!(body.hours>0)){alert('请填写工时');return}
  await post('/api/temp_tasks',body);
  renderSettings();
}
async function delTempTask(id){
  if(!confirm('删除这条临时任务？'))return;
  await post('/api/temp_tasks',{act:'del',id:id});
  renderSettings();
}
async function toggleTempTask(id,on){
  await post('/api/temp_tasks',{act:'toggle',id:id,enabled:on?1:0});
  renderSettings();
}
async function setHourRules(){
  await post('/api/hour_rules',{act:'set',team_scope_only:el('ts_only').checked?1:0});
  renderSettings();
}
async function setSkipExpired(){
  await post('/api/expired_ships',{act:'set',skip:el('skip_exp').checked?1:0});
  renderSettings();
}
async function restoreExpired(){
  await post('/api/expired_ships',{act:'restore'});
  renderSettings();
}

/* ============ 船只批量剔除 / 放回 ============ */
function shipChkAll(on){
  document.querySelectorAll('.shipchk').forEach(c=>{c.checked=!!on});
  const a=document.querySelector('.shipchk-all'); if(a)a.checked=!!on;
  shipSelCnt();shipKeep();
}
function shipChkInvert(){
  document.querySelectorAll('.shipchk').forEach(c=>{c.checked=!c.checked});
  const a=document.querySelector('.shipchk-all');
  if(a){const all=document.querySelectorAll('.shipchk').length;
        const on=document.querySelectorAll('.shipchk:checked').length;
        a.checked=(all>0&&all===on);}
  shipSelCnt();shipKeep();
}
function shipSelCnt(){
  const n=document.querySelectorAll('.shipchk:checked').length;
  const e=document.getElementById('shipsel'); if(e)e.textContent=n;
}
/* 勾选变化时，实时把该行的"参与/已剔除"标签和底色改过来（未保存前只是预览） */
function shipKeep(){
  document.querySelectorAll('.shipchk').forEach(function(c){
    const tr=c.closest('tr'); if(!tr||tr.children.length<3)return;
    const td=tr.children[2];
    if(td)td.innerHTML=c.checked
      ?'<span class="tag t-grn">参与</span>'
      :'<span class="tag t-gry">已剔除</span>';
    tr.style.background=c.checked?'':'#fafafa';
    tr.style.opacity=c.checked?'':'0.7';
  });
}
function shipPicked(){
  return Array.from(document.querySelectorAll('.shipchk:checked')).map(c=>c.dataset.s);
}
async function shipBulk(act){
  const list=shipPicked();
  if(!list.length){alert('请先勾选要操作的船');return}
  const w=(act==='exclude')?'剔除':'放回';
  if(!confirm('确定把勾选的 '+list.length+' 艘船【'+w+'】吗？'))return;
  await shipBulkPost({act:act,ships:list});
}
async function shipBulkAct(act){
  const w={exclude_all:'全部剔除',include_all:'全部放回'}[act]||act;
  if(!confirm('确定【'+w+'】吗？'+(act==='exclude_all'?'（所有在制船都会退出排产）':'')))return;
  await shipBulkPost({act:act});
}
async function shipPaste(act){
  const t=document.getElementById('ship_paste');
  const txt=((t&&t.value)||'').trim();
  if(!txt){alert('请先在文本框里粘贴船号');return}
  const w=(act==='exclude')?'剔除这些船':'只保留这些船（其余全部剔除）';
  if(!confirm('确定【'+w+'】吗？'))return;
  await shipBulkPost({act:act,ships:txt});
}
async function shipBulkPost(body){
  const r=await post('/api/ships/bulk',body);
  let msg='已处理：本次变动 '+r.changed+' 艘 ｜ 现在参与排产 '+r.in_plan+' 艘 ｜ 已剔除 '+r.excluded+' 艘';
  if(r.matched&&body.ships&&body.ships.length)
    msg='识别到 '+r.matched+' 个船号 ｜ '+msg;
  if(r.not_found&&r.not_found.length)
    msg+=' ｜ 未找到：'+r.not_found.slice(0,8).join('、')+(r.not_found.length>8?' 等 '+(r.not_found.length-8)+' 个':'');
  SHIP_MSG=msg;
  renderSettings();
}

/* ============ 复制船 ============ */
/* 点击表格里的「复制」按钮 -> 带入源船；也可在上方面板自己选 */
function cpPick(ship_no){
  const sel=el('cp_src');
  if(sel)sel.value=ship_no;
  CP_ST.src=ship_no;
  cpOnSrc();
  const box=el('cp_src');
  if(box&&box.scrollIntoView)box.scrollIntoView({block:'center'});
  cpPreview();
}
/* 把面板当前输入记进 CP_ST（重绘后恢复用） */
function cpSnap(){
  CP_ST.src=el('cp_src')?el('cp_src').value:CP_ST.src;
  CP_ST.no=el('cp_no')?el('cp_no').value:CP_ST.no;
  CP_ST.type=el('cp_type')?el('cp_type').value:CP_ST.type;
  CP_ST.mode=cpMode();
  CP_ST.dates=el('cp_dates')&&el('cp_dates').checked?1:0;
}
/* 选了源船 -> 自动填船型 + 建议新船号 */
async function cpOnSrc(){
  const src=el('cp_src')?el('cp_src').value:'';
  if(!src)return;
  CP_ST.src=src;
  // 船型直接从设置页数据里查（比读 <option> 文本稳）
  const s=(CP_SHIPS||[]).find(x=>x.ship_no===src);
  const st=(s&&s.ship_type)||(el('cp_type')?el('cp_type').value:'');
  if(st&&el('cp_type'))el('cp_type').value=st;
  CP_ST.type=st;
  await cpFillNo();
}
async function cpFillNo(){
  const st=(el('cp_type')&&el('cp_type').value)||'';
  const src=el('cp_src')?el('cp_src').value:'';
  if(!st&&!src){el('cpmsg').innerHTML='<div class="msg err">请先选源船或填船型</div>';return}
  try{
    const r=await post('/api/ship/copy',{act:'next_no',ship_type:st});
    if(r&&r.next_ship_no){
      el('cp_no').value=r.next_ship_no;
      CP_ST.no=r.next_ship_no;
      el('cpmsg').innerHTML='<div class="msg info">按船型「'+esc(st||r.ship_type)+
        '」取到下一个序号：<b>'+esc(r.next_ship_no)+'</b>（可直接改）</div>';
    }else{
      el('cpmsg').innerHTML='<div class="msg info">这个船型库里还没有同型船，'+
        '没有可参考的序号，请直接手填新船号。</div>';
    }
  }catch(e){ el('cpmsg').innerHTML='<div class="msg err">'+esc(e.message)+'</div>'; }
}
async function cpPreview(){
  cpSnap();
  const src=el('cp_src')?el('cp_src').value:'';
  if(!src){el('cpmsg').innerHTML='<div class="msg err">请先选要复制的源船</div>';return}
  const st=(el('cp_type')&&el('cp_type').value)||'';
  el('cpview').innerHTML='<div class="muted">查询中…</div>';
  try{
    const d=await post('/api/ship/copy',{act:'preview',src:src,ship_type:st});
    if(d.error){el('cpview').innerHTML='<div class="msg err">'+esc(d.error)+'</div>';return}
    const mode=cpMode();
    const useT=(mode==='template')?d.template_hours:d.src_hours;
    const useN=(mode==='template')?d.template_tasks:d.src_tasks;
    let h='<div class="tbl-wrap" style="margin-top:6px"><table><thead><tr>'+
      '<th>项目</th><th>源船 '+esc(d.src_ship_no)+'</th><th>新船（预计）</th></tr></thead><tbody>'+
      '<tr><td>船型</td><td>'+esc(d.ship_type)+'</td><td>'+esc(d.ship_type)+'</td></tr>'+
      '<tr><td>工序条数</td><td>'+d.src_tasks+' 道</td><td><b>'+useN+' 道</b>（'+
        (mode==='template'?'按船型模板展开':'照搬源船并清零')+'）</td></tr>'+
      '<tr><td>总工时</td><td>'+d.src_hours+' H（含已完成）</td><td><b>'+useT+
        ' H</b>（全部未开工）</td></tr>'+
      '<tr><td>历史报工</td><td colspan="2"><span class="t-grn">不复制</span>'+
        '（完成情况表 / 实时进度 / 排程结果都不会带过来）</td></tr>'+
      '<tr><td>时间节点</td><td colspan="2">'+
        (el('cp_dates')&&el('cp_dates').checked
          ? '沿用源船：船底启动 '+esc(fmtDate(d.dates.plan_start)||'—')+'、交检 '+
            esc(fmtDate(d.dates.plan_inspection)||'—')+' 等'
          : '<b class="t-yel">全部留空</b>，需要你在总计划里补填')+'</td></tr>'+
      '</tbody></table></div>';
    if(mode==='template'&&!d.template_tasks){
      h+='<div class="msg err" style="margin-top:6px">⚠️ 库里没有船型「'+esc(d.ship_type)+
        '」的工艺模板，无法按模板刷新工时。<br>请改用「照搬源船工序，完成量清零」，'+
        '或先导入该船型的工艺模板。</div>';
    }
    if(!d.suggest_no)h+='<div class="muted" style="margin-top:4px">（该船型库里没有同型船，'+
      '新船号请手填）</div>';
    el('cpview').innerHTML=h;
  }catch(e){ el('cpview').innerHTML='<div class="msg err">'+esc(e.message)+'</div>'; }
}
function cpMode(){
  const r=document.querySelector('input[name="cp_mode"]:checked');
  return r?r.value:'template';
}
async function cpDo(){
  cpSnap();
  const src=el('cp_src')?el('cp_src').value:'';
  const no=(el('cp_no')?el('cp_no').value:'').trim();
  const st=(el('cp_type')?el('cp_type').value:'').trim();
  const dates=el('cp_dates')&&el('cp_dates').checked?1:0;
  if(!src){alert('请先选要复制的源船');return}
  if(!no){alert('请填新船号（如 A45S009）');return}
  const mode=cpMode();
  if(!confirm('确认复制？\n\n源船：'+src+'\n新船号：'+no+'\n船型：'+(st||'沿用源船')+
    '\n工时来源：'+(mode==='template'?'按新船型模板刷新':'照搬源船并清零')+
    '\n时间节点：'+(dates?'沿用源船':'全部留空')+
    '\n\n新船会以「在制」进入总计划，历史报工不会复制。'))return;
  el('cpmsg').innerHTML='<div class="msg info">正在复制…</div>';
  try{
    const r=await post('/api/ship/copy',{act:'copy',src:src,new_ship_no:no,
      ship_type:st,task_mode:mode,copy_dates:dates});
    if(r.error){el('cpmsg').innerHTML='<div class="msg err">'+esc(r.error)+'</div>';return}
    SHIP_MSG='已复制 '+r.src_ship_no+' → '+r.ship_no+'（船型 '+r.ship_type+'）：'
      +r.tasks+' 道工序、'+r.hours+'H'+(r.no_template
        ?'　　注意：该船型没有工艺模板，工序列表为空，请改用"照搬源船"或先导入模板。':'');
    CP_ST={src:'',no:'',type:'',mode:'template',dates:0};
    renderSettings();
  }catch(e){ el('cpmsg').innerHTML='<div class="msg err">'+esc(e.message)+'</div>'; }
}

/* ============ 船底启动门禁 ============ */
async function setKeelGate(){
  const on=el('keel_gate')?el('keel_gate').checked:true;
  const rule=el('keel_rule')?el('keel_rule').value:'auto';
  const r=await post('/api/keel_gate',{act:'set',enabled:on?1:0,rule:rule});
  SHIP_MSG='船底启动门禁已'+(on?'开启':'关闭')+'，口径='
    +({auto:'实际优先',plan:'只看计划',sale:'只看实际'}[rule]||rule)
    +'；当前会被排除 '+(r.blocked_count||0)+' 艘。请到【排产与甘特图】重新排产生效。';
  renderSettings();
}
async function saveWorktime(){
  const cfg={work_sat:el('wt_sat').checked?1:0, work_sun:el('wt_sun').checked?1:0,
    ot_days_per_week:+el('wt_days').value||0, ot_hours:+el('wt_hours').value||0,
    new_overtime:el('wt_newot').checked?1:0};
  await post('/api/worktime',{cfg:cfg});
  el('wtmsg').innerHTML='<div class="msg ok">已保存，日历已按新作息重算</div>';
}
async function addHoliday(){
  if(!el('hol_date').value){alert('请先选择日期');return}
  await post('/api/holidays',{act:'add',dates:[el('hol_date').value],type:el('hol_type').value});
  el('holmsg').innerHTML='<div class="msg ok">已添加</div>';
  renderSettings();
}
async function delHoliday(){
  if(!el('hol_date').value){alert('请先选择日期');return}
  await post('/api/holidays',{act:'del',dates:[el('hol_date').value]});
  el('holmsg').innerHTML='<div class="msg ok">已删除</div>';
  renderSettings();
}
/* ---- 人力占用（支持分时间段）---- */
const MP_TEAMS=['初装','机管','电装','甲板','终装'];

function blankSeg(){
  const t={};
  MP_TEAMS.forEach(m=>t[m]={workers:0,new_workers:0});
  return {start:'',end:'',teams:t};
}
function addManpowerSeg(){
  MP_SEGS.push(blankSeg());
  renderManpowerSegs();
}
function delManpowerSeg(i){
  MP_SEGS.splice(i,1);
  if(!MP_SEGS.length)MP_SEGS.push(blankSeg());
  renderManpowerSegs();
}
function renderManpowerSegs(){
  const box=el('mp_rows');
  if(!box)return;
  if(!MP_SEGS.length)MP_SEGS.push(blankSeg());
  let h='<div class="tbl-wrap"><table><thead><tr><th style="width:70px">时间段</th>'+
        '<th style="width:130px">开始</th><th style="width:130px">结束</th>';
  MP_TEAMS.forEach(m=>h+='<th>'+m+'<br><span class="muted">人数/新人</span></th>');
  h+='<th style="width:60px"></th></tr></thead><tbody>';
  MP_SEGS.forEach((s,i)=>{
    h+='<tr><td>第'+(i+1)+'段</td>'+
       '<td><input type="date" class="mp_sd" data-i="'+i+'" value="'+(s.start||'')+'"></td>'+
       '<td><input type="date" class="mp_ed" data-i="'+i+'" value="'+(s.end||'')+'"></td>';
    MP_TEAMS.forEach(m=>{
      const v=s.teams[m]||{workers:0,new_workers:0};
      h+='<td><input type="number" style="width:46px" class="mpw" data-i="'+i+'" data-m="'+m+'" value="'+(v.workers||0)+'"> / '+
         '<input type="number" style="width:46px" class="mpn" data-i="'+i+'" data-m="'+m+'" value="'+(v.new_workers||0)+'"></td>';
    });
    h+='<td><button class="gray" onclick="delManpowerSeg('+i+')">删</button></td></tr>';
  });
  h+='</tbody></table></div>';
  box.innerHTML=h;
}
function collectManpowerSegs(){
  MP_SEGS.forEach((s,i)=>{
    const sd=document.querySelector('.mp_sd[data-i="'+i+'"]');
    const ed=document.querySelector('.mp_ed[data-i="'+i+'"]');
    if(sd)s.start=sd.value||'';
    if(ed)s.end=ed.value||'';
    const tm={};
    MP_TEAMS.forEach(m=>{
      const w=document.querySelector('.mpw[data-i="'+i+'"][data-m="'+m+'"]');
      const n=document.querySelector('.mpn[data-i="'+i+'"][data-m="'+m+'"]');
      tm[m]={workers:w?(+w.value||0):0, new_workers:n?(+n.value||0):0};
    });
    s.teams=tm;
  });
}
async function loadManpower(){
  const ship=el('mp_ship')?el('mp_ship').value:'';
  if(!ship)return;
  const r=await api('/api/manpower?ship='+encodeURIComponent(ship));
  el('mp_mode').checked=!!(r.ship&&r.ship.manpower_mode);
  const segs=(r.segs&&r.segs.length)?r.segs:[blankSeg()];
  MP_SEGS=segs.map(s=>{
    const t={};
    MP_TEAMS.forEach(m=>{
      const v=(s.teams||{})[m]||{};
      t[m]={workers:+v.workers||0, new_workers:+v.new_workers||0};
    });
    return {start:String(s.start||'').slice(0,10), end:String(s.end||'').slice(0,10), teams:t};
  });
  renderManpowerSegs();
}
async function saveManpower(){
  collectManpowerSegs();
  const segs=MP_SEGS.filter(s=>{
    return Object.keys(s.teams||{}).some(m=>(s.teams[m].workers||0)>0);
  });
  await post('/api/manpower',{act:'save',ship_no:el('mp_ship').value,
    mode:el('mp_mode').checked?1:0, segs:segs});
  el('mpmsg').innerHTML='<div class="msg ok">已保存 '+segs.length+' 个时间段。'+
    '请到【排产与甘特图】重新排产生效</div>';
}

async function saveZones(){
  const z={};
  document.querySelectorAll('.zcap').forEach(i=>{z[i.dataset.z]=+i.value||1});
  await post('/api/settings/zones',{zones:z});
  alert('区域容量已保存');
}

/* ============ 路由 ============ */
const PAGES={dashboard:renderDashboard,import:renderImport,schedule:renderSchedule,
  capacity:renderCapacity,risk:renderRisk,estimate:renderEstimate,teamplan:renderTeamPlan,
  analysis:renderAnalysis,scenario:renderScenario,settings:renderSettings};
document.querySelectorAll('#nav a').forEach(a=>{
  a.onclick=()=>{
    document.querySelectorAll('#nav a').forEach(x=>x.classList.remove('on'));
    a.classList.add('on');S.page=a.dataset.p;loadStatus();
    // 页面渲染是异步的：出错时一定要显示出来，不能让页面"点了没反应"
    try{
      Promise.resolve().then(PAGES[a.dataset.p]).catch(e=>{
        el('main').innerHTML='<h2>页面加载出错</h2><div class="msg err">'+
          (e&&e.message?e.message:String(e))+'</div>'+
          '<div class="muted">请按 F5 刷新；若反复出错，把这段提示发给维护人员</div>';
      });
    }catch(e){
      el('main').innerHTML='<div class="msg err">'+(e&&e.message?e.message:String(e))+'</div>';
    }
  };
});
async function loadStatus(){
  try{
    const st=await api('/api/status');
    S.version=st.version||S.version;
    el('status').textContent='在制船 '+st.ships+' 艘 ｜ 工序 '+st.tasks+' 道 ｜ 版本 '+(st.version||'未排产');
  }catch(e){}
}
loadStatus();
renderDashboard();
</script>
</body>
</html>
"""

# ==========================================================================
#  模块：xlsx_lite.py
# ==========================================================================
import datetime as dt
import re
import zipfile
import xml.etree.ElementTree as ET
from collections import OrderedDict

M = "http://schemas.openxmlformats.org/spreadsheetml/2006/main"
R = "http://schemas.openxmlformats.org/officeDocument/2006/relationships"
NS = {"m": M, "r": R}

EXCEL_EPOCH = dt.datetime(1899, 12, 30)

# 内置日期格式编号
_BUILTIN_DATE_FMT = {14, 15, 16, 17, 18, 19, 20, 21, 22, 45, 46, 47}


# ============================================================ 工具
def col_index(ref: str) -> int:
    """单元格地址 'AB12' -> 列索引 27（从 0 开始）"""
    n = 0
    for ch in ref:
        if ch.isalpha():
            n = n * 26 + (ord(ch.upper()) - 64)
        else:
            break
    return n - 1


def col_name(idx: int) -> str:
    """列索引 0 -> 'A'，26 -> 'AA'"""
    s = ""
    idx += 1
    while idx:
        idx, r = divmod(idx - 1, 26)
        s = chr(65 + r) + s
    return s


def serial_to_datetime(serial: float):
    """Excel 日期序列号 -> datetime（基准 1899-12-30）"""
    try:
        return EXCEL_EPOCH + dt.timedelta(days=float(serial))
    except (OverflowError, ValueError):
        return None


# ============================================================ 读取
def _read_shared_strings(z: zipfile.ZipFile) -> list:
    if "xl/sharedStrings.xml" not in z.namelist():
        return []
    root = ET.fromstring(z.read("xl/sharedStrings.xml"))
    out = []
    for si in root.findall(f"{{{M}}}si"):
        # 拼接所有 <t>（处理富文本被拆成多个 run 的情况）
        out.append("".join(t.text or "" for t in si.iter(f"{{{M}}}t")))
    return out


def _read_date_styles(z: zipfile.ZipFile) -> set:
    """返回"哪些样式编号是日期格式"的集合，用于区分数字与日期"""
    if "xl/styles.xml" not in z.namelist():
        return set()
    try:
        root = ET.fromstring(z.read("xl/styles.xml"))
    except ET.ParseError:
        return set()
    custom = {}
    for nf in root.iter(f"{{{M}}}numFmt"):
        try:
            custom[int(nf.get("numFmtId"))] = nf.get("formatCode", "") or ""
        except (TypeError, ValueError):
            continue
    date_styles = set()
    cellxfs = root.find(f"{{{M}}}cellXfs")
    if cellxfs is None:
        return date_styles
    for i, xf in enumerate(cellxfs.findall(f"{{{M}}}xf")):
        try:
            fid = int(xf.get("numFmtId", "0"))
        except ValueError:
            continue
        code = custom.get(fid, "")
        if fid in _BUILTIN_DATE_FMT:
            date_styles.add(i)
        elif code:
            # 去掉转义和字符串字面量后再判断是否含日期占位符
            cleaned = re.sub(r'\[[^\]]*\]|"[^"]*"|\\.', "", code)
            if re.search(r"[ymdhs]", cleaned, re.IGNORECASE):
                date_styles.add(i)
    return date_styles


def _read_sheet(z: zipfile.ZipFile, target: str, sst: list, date_styles: set,
                max_rows: int = 0) -> list:
    """读取一个工作表的全部单元格（返回二维列表，空单元格为 None）"""
    try:
        root = ET.fromstring(z.read(target))
    except (KeyError, ET.ParseError):
        return []
    data = root.find(f"{{{M}}}sheetData")
    if data is None:
        return []

    rows = []
    max_col = 0
    for row in data.findall(f"{{{M}}}row"):
        try:
            r_idx = int(row.get("r", len(rows) + 1))
        except ValueError:
            r_idx = len(rows) + 1
        while len(rows) < r_idx - 1:
            rows.append([])
        cells = {}
        for c in row.findall(f"{{{M}}}c"):
            ref = c.get("r", "")
            ci = col_index(ref) if ref else len(cells)
            t = c.get("t")
            s = c.get("s")
            v = c.find(f"{{{M}}}v")
            isel = c.find(f"{{{M}}}is")
            val = None

            if t == "inlineStr":
                if isel is not None:
                    val = "".join(x.text or "" for x in isel.iter(f"{{{M}}}t"))
            elif v is not None:
                raw = v.text
                if raw is None:
                    val = None
                elif t == "s":
                    try:
                        val = sst[int(raw)] if int(raw) < len(sst) else ""
                    except (ValueError, IndexError):
                        val = ""
                elif t == "str":
                    val = raw
                elif t == "b":
                    val = (raw == "1")
                elif t == "e":
                    val = "#" + raw          # #VALUE! / #REF! 等错误值
                else:                         # 数字 n
                    try:
                        f = float(raw)
                        if s is not None:
                            try:
                                si = int(s)
                            except ValueError:
                                si = -1
                            if si in date_styles and 1 < f < 80000:
                                val = serial_to_datetime(f)
                            else:
                                val = int(f) if f == int(f) else f
                        else:
                            val = int(f) if f == int(f) else f
                    except ValueError:
                        val = raw
            if val is not None and val != "":
                cells[ci] = val
                max_col = max(max_col, ci + 1)
        line = [cells.get(i) for i in range(max_col)]
        rows.append(line)
        if max_rows and len(rows) >= max_rows:
            break

    # 补齐每行长度
    for r in rows:
        if len(r) < max_col:
            r.extend([None] * (max_col - len(r)))
    return rows


def read_workbook(path: str) -> "OrderedDict[str, list]":
    """
    读取 xlsx，返回 OrderedDict：{工作表名: 二维列表}

    单元格值类型：str / int / float / datetime / bool / None
    """
    z = zipfile.ZipFile(path)
    try:
        sst = _read_shared_strings(z)
        date_styles = _read_date_styles(z)

        sheets = []
        try:
            wb = ET.fromstring(z.read("xl/workbook.xml"))
            rels = ET.fromstring(z.read("xl/_rels/workbook.xml.rels"))
            relmap = {r.get("Id"): r.get("Target") for r in rels}
            for sh in wb.find(f"{{{M}}}sheets").findall(f"{{{M}}}sheet"):
                rid = sh.get(f"{{{R}}}id")
                target = relmap.get(rid, "")
                if target.startswith("/"):
                    target = target.lstrip("/")
                elif not target.startswith("xl/"):
                    target = "xl/" + target
                sheets.append((sh.get("name") or f"Sheet{len(sheets)+1}", target))
        except (KeyError, ET.ParseError, AttributeError):
            # 结构异常时退化为直接扫描 worksheets 目录
            for n in sorted(z.namelist()):
                if n.startswith("xl/worksheets/sheet") and n.endswith(".xml"):
                    sheets.append((n.split("/")[-1], n))

        result = OrderedDict()
        for name, target in sheets:
            result[name] = _read_sheet(z, target, sst, date_styles)
        return result
    finally:
        z.close()


def read_workbook_lite(path: str, sheet_names=None, max_rows: int = 0):
    """只读指定工作表（大文件省内存）。sheet_names=None 表示全部。"""
    z = zipfile.ZipFile(path)
    try:
        sst = _read_shared_strings(z)
        date_styles = _read_date_styles(z)
        wb = ET.fromstring(z.read("xl/workbook.xml"))
        rels = ET.fromstring(z.read("xl/_rels/workbook.xml.rels"))
        relmap = {r.get("Id"): r.get("Target") for r in rels}
        result = OrderedDict()
        for sh in wb.find(f"{{{M}}}sheets").findall(f"{{{M}}}sheet"):
            name = sh.get("name")
            if sheet_names and name not in sheet_names:
                continue
            target = relmap.get(sh.get(f"{{{R}}}id"), "")
            if not target.startswith("xl/"):
                target = "xl/" + target.lstrip("/")
            result[name] = _read_sheet(z, target, sst, date_styles, max_rows=max_rows)
        return result
    finally:
        z.close()


# ============================================================ 写入
def _esc(s) -> str:
    s = "" if s is None else str(s)
    return (s.replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;")
             .replace('"', "&quot;"))


def _cell_xml(value, col: int, row: int) -> str:
    if value is None or value == "":
        return ""
    ref = f"{col_name(col)}{row}"
    if isinstance(value, bool):
        return f'<c r="{ref}" t="b"><v>{1 if value else 0}</v></c>'
    if isinstance(value, (int, float)):
        return f'<c r="{ref}"><v>{value}</v></c>'
    if isinstance(value, (dt.date, dt.datetime)):
        txt = value.strftime("%Y-%m-%d") if isinstance(value, dt.datetime) else value.isoformat()
        return f'<c r="{ref}" t="inlineStr"><is><t>{_esc(txt)}</t></is></c>'
    return f'<c r="{ref}" t="inlineStr"><is><t xml:space="preserve">{_esc(value)}</t></is></c>'


def write_workbook(path: str, sheets: list) -> str:
    """
    写出一个 xlsx 文件（多工作表），Excel / WPS 可直接打开。

    :param path:   输出路径
    :param sheets: [(表名, 二维列表), ...]
    """
    n = len(sheets)

    content_types = ['<?xml version="1.0" encoding="UTF-8" standalone="yes"?>',
                     '<Types xmlns="http://schemas.openxmlformats.org/package/2006/content-types">',
                     '<Default Extension="rels" ContentType="application/vnd.openxmlformats-package.relationships+xml"/>',
                     '<Default Extension="xml" ContentType="application/xml"/>',
                     '<Override PartName="/xl/workbook.xml" ContentType="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet.main+xml"/>']
    for i in range(n):
        content_types.append(
            f'<Override PartName="/xl/worksheets/sheet{i+1}.xml" '
            f'ContentType="application/vnd.openxmlformats-officedocument.spreadsheetml.worksheet+xml"/>')
    content_types.append("</Types>")

    root_rels = ('<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'
                 '<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">'
                 '<Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/officeDocument" Target="xl/workbook.xml"/>'
                 '</Relationships>')

    sheet_tags = "".join(
        f'<sheet name="{_esc(name)}" sheetId="{i+1}" r:id="rId{i+1}"/>'
        for i, (name, _) in enumerate(sheets))
    workbook = ('<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'
                f'<workbook xmlns="{M}" xmlns:r="{R}"><sheets>{sheet_tags}</sheets></workbook>')

    wb_rels = ['<?xml version="1.0" encoding="UTF-8" standalone="yes"?>',
               '<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">']
    for i in range(n):
        wb_rels.append(
            f'<Relationship Id="rId{i+1}" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/worksheet"'
            f' Target="worksheets/sheet{i+1}.xml"/>')
    wb_rels.append("</Relationships>")

    with zipfile.ZipFile(path, "w", zipfile.ZIP_DEFLATED) as z:
        z.writestr("[Content_Types].xml", "".join(content_types))
        z.writestr("_rels/.rels", root_rels)
        z.writestr("xl/workbook.xml", workbook)
        z.writestr("xl/_rels/workbook.xml.rels", "".join(wb_rels))
        for i, (_name, rows) in enumerate(sheets):
            body = ['<?xml version="1.0" encoding="UTF-8" standalone="yes"?>',
                    f'<worksheet xmlns="{M}"><sheetData>']
            for ri, row in enumerate(rows, start=1):
                cells = "".join(_cell_xml(v, ci, ri) for ci, v in enumerate(row))
                if cells:
                    body.append(f'<row r="{ri}">{cells}</row>')
            body.append("</sheetData></worksheet>")
            z.writestr(f"xl/worksheets/sheet{i+1}.xml", "".join(body))
    return path

# ==========================================================================
#  模块：aps_engine.py
# ==========================================================================
import datetime as dt
import json
import os
import re
import sqlite3
from collections import defaultdict, Counter


# ==================================================================
# 一、常量与默认配置
# ==================================================================
BASE_DIR = os.path.dirname(os.path.abspath(__file__))


def _pick_data_dir():
    """
    选择数据目录：
      1. 优先放在本程序所在文件夹（绿色软件习惯，数据跟着文件夹走）
      2. 若该位置不可写（程序放在 U 盘、网盘同步目录、系统目录等），
         自动回退到用户目录 %LOCALAPPDATA%\\ShipAPS，保证一定能跑起来
    """
    import tempfile
    for base in (BASE_DIR, os.path.join(os.environ.get("LOCALAPPDATA", ""), "ShipAPS")):
        try:
            d = os.path.join(base, "data")
            os.makedirs(d, exist_ok=True)
            probe = os.path.join(d, ".write_test")
            with open(probe, "w", encoding="utf-8") as f:
                f.write("1")
            os.remove(probe)
            return d, base
        except OSError:
            continue
    # 最后兜底：临时目录（至少能跑）
    d = os.path.join(tempfile.gettempdir(), "ShipAPS_data")
    os.makedirs(d, exist_ok=True)
    return d, BASE_DIR


DATA_DIR, DATA_BASE = _pick_data_dir()
RAW_DIR = os.path.join(DATA_DIR, "raw")
OUT_DIR = os.path.join(DATA_DIR, "output")
DB_PATH = os.path.join(DATA_DIR, "aps.db")
for _d in (DATA_DIR, RAW_DIR, OUT_DIR):
    os.makedirs(_d, exist_ok=True)

TEAM_CODES = ["初装", "机管", "电装", "甲板", "终装"]
TEAM_ORDER = {t: i for i, t in enumerate(TEAM_CODES)}

STATUS_DONE, STATUS_DOING, STATUS_TODO = "已完成", "进行中", "未开始"
STATUS_FROZEN, STATUS_ABNORMAL = "冻结", "异常"
STATUS_ALIAS = {
    "已完工": STATUS_DONE, "已完成": STATUS_DONE, "完工": STATUS_DONE, "完成": STATUS_DONE,
    "结束": STATUS_DONE, "进行中": STATUS_DOING, "正在生产": STATUS_DOING,
    "生产中": STATUS_DOING, "施工中": STATUS_DOING, "部分完成": STATUS_DOING,
    "未开始": STATUS_TODO, "未开工": STATUS_TODO, "待作业": STATUS_TODO, "未作业": STATUS_TODO,
    "冻结": STATUS_FROZEN, "暂停": STATUS_FROZEN, "暂缓": STATUS_FROZEN,
    "异常": STATUS_ABNORMAL, "缺料": STATUS_ABNORMAL, "受阻": STATUS_ABNORMAL,
}
PRIORITY_MAP = {"紧急": 0, "高": 1, "正常": 2, "低": 3, "暂缓": 4, "冻结": 9}

SHIP_TYPE_ALIAS = {"A360": "A36S", "A3601": "A36S", "A360S": "A36S"}
TEAM_ALIAS = {
    "机装": "机管", "管装": "机管", "机电": "机管", "轮机": "机管", "管道": "机管",
    "内装": "终装", "木装": "终装", "家装": "终装", "油装": "终装", "居装": "终装",
    "木工": "终装", "电气": "电装", "电工": "电装", "舾装": "甲板", "外装": "甲板",
    "钢装": "初装", "船体": "初装",
    # 单船实时进度表里出现的班组写法
    "初装一组": "初装", "机管一组": "机管", "电装一组": "电装",
    "甲板一组": "甲板", "终装一组": "终装", "初装二组": "初装",
    "机管二组": "机管", "电装二组": "电装", "甲板二组": "甲板", "终装二组": "终装",
    "胶修": "终装", "油漆": "终装",
    # 总装之前的上游班组（铺糊/泡沫/脱模切割/CNC），不属于总装五班组，
    # 单独归为"前置"，不占用总装产能，但参与完成率统计
    "泡沫": "前置", "脱模切割": "前置", "CNC": "前置", "铺糊": "前置",
}

# 上游（总装之前）班组：不参与总装排产，只用于判断完成情况
PRE_TEAMS = {"前置"}
# "不锈钢"组（A36MC/A46C/A47MC 周期表特有）2026-10-29 PMC 确认：算甲板的。
# 模板导入时会自动映射成"甲板"（见 TEAM_REMAP），历史数据也已刷成甲板。

# 周期表里的特殊组名 -> 归属的排产班组（导入时自动映射）
TEAM_REMAP = {"不锈钢": "甲板"}

# 单船实时进度表里"视为已完成"的状态（加工中也算完成，且不再计剩余工时）
DONE_STATUS_RT = {"已完工", "已完成", "加工中", "生产中", "施工中", "部分完成", ""}
# 排布计划详情（PMC 排布单）里"视为已完成"的状态。
# 用户口径：显示 已完成/已完工/加工中/作业中 这类就算该工序完工、不计入工时。
# "进行中"与"加工中"同义，一并纳入；如需收紧，删掉"进行中"即可。
PLAN_DONE_STATUS = {"已完成", "已完工", "加工中", "作业中", "进行中", "生产中", "施工中",
                    "部分完成", "完工", "完成", "X", "x"}
# 排布计划详情的表头别名（列名不规范时也能对上）
_PLAN_COLS = {
    "船号": ["船号", "船只", "船"],
    "工序编码": ["工序编码", "工序代码", "工序编号"],
    "工序名称": ["工序名称", "工序"],
    "班组": ["所属班组", "班组", "排布班组"],
    "工段": ["工段属性", "工段"],
    "定额工时": ["定额工时（H）", "定额工时(H)", "定额工时"],
    "剩余工时": ["剩余工时（H）", "剩余工时(H)", "剩余工时"],
    "状态": ["状态"],
}
DEFAULT_ZONE_CAPACITY = {"机舱": 1, "发动机舱": 1, "卫生间": 1, "驾驶台": 1, "主舱": 2,
                         "舱室": 2, "船底": 2, "甲板": 2, "硬顶": 1, "船艏": 2, "船艉": 2,
                         "左舷": 2, "右舷": 2, "过道": 2, "船下": 1, "全船": 5}
DEFAULT_CONFLICTS = [("打磨", "家具安装", True, "禁止"), ("打磨", "玻璃安装", True, "禁止"),
                     ("开孔", "电气调试", True, "禁止"), ("切割", "清洁", True, "禁止"),
                     ("粘接", "打磨", True, "限制"), ("打胶", "打磨", True, "限制"),
                     ("吊装", "船下作业", False, "禁止"), ("吊装", "过道作业", False, "禁止")]
TASK_TYPE_KEYWORDS = {
    "打磨": ["打磨", "砂纸", "磨"], "开孔": ["开孔", "钻孔", "打孔"],
    "切割": ["切割", "裁切", "CNC"], "粘接": ["粘接", "粘结", "糊制", "手糊"],
    "打胶": ["打胶", "填缝", "密封胶", "胶衣"],
    "家具安装": ["家具", "柜", "床", "沙发", "座椅", "软包", "地毯"],
    "玻璃安装": ["玻璃", "窗", "有机玻璃"], "电气调试": ["调试", "通电", "试验", "报验"],
    "清洁": ["清洁", "清洗", "清扫"], "吊装": ["吊装", "吊船", "吊运", "起吊", "压甲板", "合拢"],
    "船下作业": ["船下", "船底外", "下水"], "过道作业": ["过道", "通道"],
}
DEFAULT_HOLIDAYS = ["2026-10-01", "2026-10-02", "2026-10-03", "2026-10-04", "2026-10-05",
                    "2026-10-06", "2026-10-07", "2027-01-01"]

DEFAULT_TEAMS = {
    "初装": dict(skilled=12, new=3, sh=11.0, nh=4.0, ship_max=6, ship_max_u=7),
    "机管": dict(skilled=10, new=2, sh=11.0, nh=4.0, ship_max=4, ship_max_u=5),
    "电装": dict(skilled=10, new=2, sh=11.0, nh=4.0, ship_max=4, ship_max_u=5),
    "甲板": dict(skilled=12, new=3, sh=11.0, nh=4.0, ship_max=5, ship_max_u=6),
    "终装": dict(skilled=12, new=4, sh=11.0, nh=4.0, ship_max=5, ship_max_u=6),
}

# ---------------------------------------------------------------- 作息与加班默认配置
# 这些是 PMC 最常调整的参数，界面里可以直接改；改完自动重算日历与产能
WORKTIME_DEFAULTS = {
    "work_sat": 0,            # 周六上班（用户要求：默认不上班）
    "work_sun": 0,            # 周日上班（用户要求：默认不上班）
    "ot_days_per_week": 3,    # 每周加班天数（用户要求：默认 3 天）
    "ot_hours": 3.0,          # 每次加班小时（用户要求：默认 3 小时）
    "new_overtime": 0,        # 新人是否参与加班（用户要求：默认不加班）
    "sh_per_day": 11.0,       # 熟练工每日有效工时
    "nh_per_day": 4.0,        # 新人每日有效工时
}


def get_setting(conn, key, default=None):
    r = conn.execute("SELECT value FROM settings WHERE key=?", (key,)).fetchone()
    return r["value"] if r else default


def set_setting(conn, key, value):
    conn.execute("INSERT OR REPLACE INTO settings(key,value) VALUES(?,?)", (key, str(value)))
    conn.commit()


def prod_days_between(conn, a, b):
    """a~b 之间（含两端）的生产日天数；日历没建时按工作日估算"""
    try:
        n = conn.execute(
            "SELECT COUNT(*) c FROM calendar WHERE cal_date BETWEEN ? AND ? AND is_prod=1",
            (a.isoformat(), b.isoformat())).fetchone()["c"]
        if n:
            return int(n)
    except Exception:
        pass
    n, d = 0, a
    while d <= b:
        if d.weekday() < 5:
            n += 1
        d += dt.timedelta(days=1)
    return max(1, n)


def temp_task_daily_hours(conn, team, d, today=None):
    """
    某班组在 d 这天被【临时任务】占用的工时。

    起止都给了 -> 这段时间内按生产日均摊；只给开始 -> 往后 30 天摊；
    只给结束 -> 从今天摊到结束日；都没给 -> 算在今天。
    """
    today = today or dt.date.today()
    tot = 0.0
    try:
        rows = conn.execute(
            "SELECT start_date,end_date,hours FROM temp_tasks "
            "WHERE team=? AND COALESCE(enabled,1)=1", (team,)).fetchall()
    except Exception:
        return 0.0
    for r in rows:
        try:
            h = float(r["hours"] or 0)
        except (TypeError, ValueError):
            continue
        if h <= 0:
            continue
        s = to_date(r["start_date"])
        e = to_date(r["end_date"])
        if s is None and e is None:
            if d == today:
                tot += h
            continue
        if s is None:
            s = today if (e and today <= e) else (e or today)
        if e is None:
            e = s + dt.timedelta(days=30)
        if e < s:
            s, e = e, s
        if not (s <= d <= e):
            continue
        tot += h / max(1, prod_days_between(conn, s, e))
    return tot


def get_worktime_config(conn) -> dict:
    """读取作息/加班配置（班组级别的作息以 teams 表为准，这里只存全局默认）"""
    cfg = dict(WORKTIME_DEFAULTS)
    for k in cfg:
        v = get_setting(conn, "worktime_" + k)
        if v is not None:
            try:
                cfg[k] = float(v) if k in ("ot_hours", "sh_per_day", "nh_per_day") else int(float(v))
            except ValueError:
                pass
    return cfg


def save_worktime_config(conn, cfg: dict) -> dict:
    for k, v in (cfg or {}).items():
        if k in WORKTIME_DEFAULTS:
            set_setting(conn, "worktime_" + k, v)
    return get_worktime_config(conn)

log_lines = []


def log(msg: str):
    line = f"{dt.datetime.now():%H:%M:%S} {msg}"
    log_lines.append(line)
    if len(log_lines) > 400:
        log_lines.pop(0)


# ==================================================================
# 二、容错解析工具
# ==================================================================
def norm_text(v) -> str:
    if v is None:
        return ""
    s = str(v).replace("\u3000", " ").replace("\n", " ").replace("\r", " ")
    return re.sub(r"\s+", " ", s).strip()


def to_number(v, default=0.0):
    """数字容错解析。Excel 把工时存成日期（8H -> 1900-01-08）时自动还原成小时数。"""
    if v is None:
        return default
    if isinstance(v, bool):
        return 1.0 if v else 0.0
    if isinstance(v, dt.datetime):
        # 伪日期工时：年份极早（1900 附近）说明这个"日期"其实是小时数
        if v.year <= 1905:
            return round((v.date() - dt.date(1899, 12, 31)).days + v.hour / 24 + v.minute / 1440, 2)
        return default
    if isinstance(v, (int, float)):
        return float(v)
    s = norm_text(v)
    if s in {"", "-", "#VALUE!", "#REF!", "#NAME?", "#DIV/0!", "#N/A", "N/A"}:
        return default
    try:
        return float(s.replace(",", ""))
    except ValueError:
        m = re.search(r"-?\d+(\.\d+)?", s.replace(",", ""))
        return float(m.group(0)) if m else default


def to_date(v):
    """日期容错解析：datetime / Excel 序列号 / 各种字符串写法 -> date"""
    if v is None:
        return None
    if isinstance(v, dt.datetime):
        return None if v.year <= 1905 else v.date()
    if isinstance(v, dt.date):
        return v
    if isinstance(v, bool):
        return None
    s = norm_text(v)
    if s in {"", "-", "#VALUE!", "#REF!", "#NAME?", "#N/A", "N/A", "无"}:
        return None
    if isinstance(v, (int, float)):
        f = float(v)
        if 2 <= f <= 80000:                      # Excel 日期序列号
            try:
                return (dt.date(1899, 12, 30) + dt.timedelta(days=int(f)))
            except (OverflowError, ValueError):
                return None
        if 19000101 <= int(f) <= 21001231:       # 20260923 这种伪日期数字
            s = str(int(f))
        else:
            return None
    s2 = re.split(r"[ T]", s)[0]
    for p in ("%Y-%m-%d", "%Y/%m/%d", "%Y.%m.%d", "%Y%m%d", "%y-%m-%d",
              "%m-%d", "%m/%d", "%Y年%m月%d日", "%m月%d日"):
        try:
            d = dt.datetime.strptime(s2, p).date()
            if d.year == 1900 and "%Y" not in p and "%y" not in p:
                d = d.replace(year=dt.date.today().year)
            return d
        except ValueError:
            continue
    m = re.fullmatch(r"(\d{4})(\d{2})(\d{2})", s2)
    if m:
        try:
            return dt.date(int(m.group(1)), int(m.group(2)), int(m.group(3)))
        except ValueError:
            return None
    return None


def guess_ship_type(ship_no: str) -> str:
    s = norm_text(ship_no).upper()
    m = re.match(r"^([A-Z]+[0-9]*[A-Z]*?)(\d{2,})$", s)
    if m:
        return m.group(1)
    m = re.match(r"^([A-Za-z]+)(\d+)$", s)
    return m.group(1).upper() if m else ""


def resolve_ship_type(ship_no: str) -> str:
    t = guess_ship_type(ship_no)
    return SHIP_TYPE_ALIAS.get(t, t)


def normalize_team(v) -> str:
    s = norm_text(v)
    for k in TEAM_CODES:
        if k in s:
            return k
    for k, val in TEAM_ALIAS.items():
        if k in s:
            return val
    return s


def guess_task_type(name: str) -> str:
    if not name:
        return "通用"
    for t, kws in TASK_TYPE_KEYWORDS.items():
        for kw in kws:
            if kw in name:
                return t
    return "通用"


def parse_day_no(v, default=1) -> int:
    if isinstance(v, (int, float)) and not isinstance(v, bool):
        try:
            return max(1, int(v))
        except (ValueError, OverflowError):
            return default
    m = re.search(r"(\d+)", norm_text(v))
    return max(1, int(m.group(1))) if m else default


def row_text(row) -> str:
    return "".join("" if c is None else str(c) for c in row)


def find_col(headers, keyword, nth=1) -> int:
    """在表头里找关键字第 nth 次出现的列索引，找不到返回 -1"""
    cnt = 0
    for i, h in enumerate(headers):
        if h is not None and keyword in str(h):
            cnt += 1
            if cnt == nth:
                return i
    return -1


def detect_header(rows, keywords, scan=15) -> int:
    """在前 scan 行里找包含全部关键字的表头行"""
    best, best_score = -1, 0
    for i in range(min(scan, len(rows))):
        t = row_text(rows[i])
        score = sum(1 for k in keywords if k in t)
        if score > best_score:
            best, best_score = i, score
    return best if best_score >= max(1, len(keywords) - 1) else -1


def is_blank_row(row) -> bool:
    return all(c is None or str(c).strip() == "" for c in row)


def cell(row, idx):
    if idx is None or idx < 0 or idx >= len(row):
        return None
    v = row[idx]
    return None if (v is None or str(v).strip() == "") else v


def weekday_cn(d) -> str:
    return "周" + "一二三四五六日"[d.weekday()] if d else ""


# ==================================================================
# 三、数据库
# ==================================================================
SCHEMA = """
CREATE TABLE IF NOT EXISTS ships(
 ship_no TEXT PRIMARY KEY, ship_type TEXT, destination TEXT,
 sale_start DATE, sale_demould DATE, plan_start DATE, plan_demould DATE,
 plan_deck_bond DATE, plan_hardtop DATE, plan_inspection DATE,
 priority INTEGER DEFAULT 2, manual_priority TEXT DEFAULT '正常',
 must_finish_on_time INTEGER DEFAULT 0, allow_delay INTEGER DEFAULT 1,
 status TEXT DEFAULT '在制', current_stage TEXT, predicted_finish DATE,
 delay_days INTEGER DEFAULT 0,
 risk_level TEXT, done_hours REAL DEFAULT 0, remaining_hours REAL DEFAULT 0,
 progress REAL DEFAULT 0, bottleneck TEXT, delay_reason TEXT, remarks TEXT,
 fixed_workers INTEGER DEFAULT 0, fixed_new INTEGER DEFAULT 0);
    CREATE TABLE IF NOT EXISTS temp_tasks(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        team TEXT,                 -- 班组
        start_date DATE,           -- 可空
        end_date DATE,             -- 可空
        hours REAL,                -- 临时增加的工时总量
        note TEXT,
        enabled INTEGER DEFAULT 1
    );
CREATE TABLE IF NOT EXISTS teams(
 team_code TEXT PRIMARY KEY, skilled INTEGER, new_workers INTEGER,
 sh_per_day REAL, nh_per_day REAL, work_sat INTEGER DEFAULT 1, work_sun INTEGER DEFAULT 0,
 overtime REAL DEFAULT 0, ship_max INTEGER DEFAULT 5, ship_max_u INTEGER DEFAULT 6,
 max_per_task INTEGER DEFAULT 4, sort_order INTEGER DEFAULT 0,
 ot_days_per_week INTEGER DEFAULT 3, ot_hours REAL DEFAULT 3.0,
 new_overtime INTEGER DEFAULT 0);
CREATE TABLE IF NOT EXISTS ship_types(
 type_code TEXT PRIMARY KEY, cycle_days INTEGER, total_hours REAL,
 process_count INTEGER, source_file TEXT);
CREATE TABLE IF NOT EXISTS task_templates(
 id INTEGER PRIMARY KEY AUTOINCREMENT, ship_type TEXT, process_code TEXT,
 process_name TEXT, team TEXT, section TEXT, standard_hours REAL,
 earliest_day INTEGER, latest_day INTEGER, zone TEXT, task_type TEXT,
 milestone TEXT DEFAULT '', predecessor_text TEXT, material_text TEXT);
CREATE TABLE IF NOT EXISTS task_dependencies(
 id INTEGER PRIMARY KEY AUTOINCREMENT, ship_type TEXT, process_code TEXT,
 predecessor_code TEXT, lag_days INTEGER DEFAULT 0, source TEXT);
CREATE TABLE IF NOT EXISTS tasks(
 id INTEGER PRIMARY KEY AUTOINCREMENT, ship_no TEXT, ship_type TEXT, process_code TEXT,
 process_name TEXT, team TEXT, section TEXT, zone TEXT, task_type TEXT,
 planned_hours REAL DEFAULT 0, done_hours REAL DEFAULT 0, remaining_hours REAL DEFAULT 0,
 status TEXT DEFAULT '未开始', progress REAL DEFAULT 0, confidence REAL DEFAULT 1,
 actual_finish DATE, earliest_day INTEGER DEFAULT 1, latest_day INTEGER DEFAULT 1,
 min_w INTEGER DEFAULT 1, max_w INTEGER DEFAULT 4, critical INTEGER DEFAULT 0,
 milestone TEXT, material_ready INTEGER DEFAULT 1, source TEXT DEFAULT '模板', remarks TEXT,
 UNIQUE(ship_no, process_code));
CREATE TABLE IF NOT EXISTS calendar(
 cal_date DATE PRIMARY KEY, day_type TEXT, is_prod INTEGER DEFAULT 1,
 normal_hours REAL DEFAULT 11, overtime REAL DEFAULT 0, remarks TEXT);
CREATE TABLE IF NOT EXISTS team_calendar(
 id INTEGER PRIMARY KEY AUTOINCREMENT, cal_date DATE, team TEXT, is_working INTEGER DEFAULT 1,
 leave_cnt INTEGER DEFAULT 0, borrow_in INTEGER DEFAULT 0, borrow_out INTEGER DEFAULT 0,
 temp_add INTEGER DEFAULT 0, overtime REAL DEFAULT 0, UNIQUE(cal_date, team));
CREATE TABLE IF NOT EXISTS actual_progress(
 id INTEGER PRIMARY KEY AUTOINCREMENT, team TEXT, ship_no TEXT, process_code TEXT,
 process_name TEXT, section TEXT, hours REAL, week_finish DATE, actual_finish DATE,
 raw_status TEXT, status TEXT, not_done_reason TEXT, confidence REAL,
 source_file TEXT, source_sheet TEXT);
CREATE TABLE IF NOT EXISTS ship_manpower(
 id INTEGER PRIMARY KEY AUTOINCREMENT, ship_no TEXT, team TEXT,
 workers INTEGER DEFAULT 0, new_workers INTEGER DEFAULT 0,
 start_date DATE, end_date DATE, note TEXT, seg INTEGER DEFAULT 0);
CREATE TABLE IF NOT EXISTS realtime_progress(
 id INTEGER PRIMARY KEY AUTOINCREMENT, ship_no TEXT, process_code TEXT,
 process_name TEXT, team TEXT, section TEXT, quota_hours REAL, remain_hours REAL,
 start_time TEXT, finish_time TEXT, raw_status TEXT, done INTEGER DEFAULT 0,
 source_file TEXT, UNIQUE(ship_no, process_code));
CREATE TABLE IF NOT EXISTS plan_detail(
 id INTEGER PRIMARY KEY AUTOINCREMENT, ship_no TEXT, process_code TEXT,
 process_name TEXT, team TEXT, section TEXT, quota_hours REAL, remain_hours REAL,
 plan_finish TEXT, actual_finish TEXT, raw_status TEXT, done INTEGER DEFAULT 0,
 source_file TEXT, UNIQUE(ship_no, process_code));
-- 作废船黑名单：用户在界面上"叉掉"的船，删掉后不允许再被主计划/班组表/进度表
-- 自动复活（否则点一次"导入"它就又回来了，用户反馈"A70L007我已经叉掉却还在排产里"）
CREATE TABLE IF NOT EXISTS excluded_ships(
 ship_no TEXT PRIMARY KEY, reason TEXT, created_at TEXT);
-- 船期延期设置（产能分析页表单写入）：排产时按 delay_days 顺延该船 plan_inspection
CREATE TABLE IF NOT EXISTS ship_delays(
 ship_no TEXT PRIMARY KEY, delay_days REAL DEFAULT 0, reason TEXT, updated_at TEXT);
CREATE TABLE IF NOT EXISTS zones(
 zone_code TEXT PRIMARY KEY, zone_name TEXT, max_teams INTEGER DEFAULT 2);
CREATE TABLE IF NOT EXISTS task_conflicts(
 id INTEGER PRIMARY KEY AUTOINCREMENT, ta TEXT, tb TEXT, same_zone INTEGER DEFAULT 1,
 scope TEXT DEFAULT 'ship',
 level TEXT DEFAULT '禁止', enabled INTEGER DEFAULT 1);
CREATE TABLE IF NOT EXISTS milestones(
 id INTEGER PRIMARY KEY AUTOINCREMENT, ship_type TEXT, code TEXT, name TEXT,
 plan_day INTEGER, is_gate INTEGER DEFAULT 1);
CREATE TABLE IF NOT EXISTS schedule_versions(
 id INTEGER PRIMARY KEY AUTOINCREMENT, version_no TEXT UNIQUE, created_at TEXT,
 algorithm TEXT, params TEXT, metrics TEXT, is_active INTEGER DEFAULT 0);
CREATE TABLE IF NOT EXISTS schedule_result(
 id INTEGER PRIMARY KEY AUTOINCREMENT, version_no TEXT, work_date DATE, ship_no TEXT,
 ship_type TEXT, process_code TEXT, process_name TEXT, team TEXT, workers REAL,
 planned_hours REAL, zone TEXT, priority INTEGER, reason TEXT);
CREATE TABLE IF NOT EXISTS data_quality(
 id INTEGER PRIMARY KEY AUTOINCREMENT, src TEXT, sheet TEXT, row_no INTEGER,
 col TEXT, issue TEXT, detail TEXT, raw TEXT, severity TEXT);
CREATE TABLE IF NOT EXISTS settings(key TEXT PRIMARY KEY, value TEXT);
-- 「工序工位」文件里的工时总表：按船号 × 五个作业组记录总工时。
-- 数据来源：设计开发二部下发的《xxx 工序工位 REVx.xlsx》→「工时总表」工作表。
-- hours_raw 保留该船型行 17 个班组列的原始明细（JSON），保证原始数据不丢。
CREATE TABLE IF NOT EXISTS ship_group_hours(
 ship_no TEXT, ship_type TEXT, group_code TEXT,
 hours REAL DEFAULT 0, hours_raw TEXT,
 src_file TEXT, rev TEXT, updated_at TEXT,
 UNIQUE(ship_no, group_code));
-- 工序工位文件里识别到的船号台账：先落库待确认，用户在界面上手动决定是否加入总计划。
CREATE TABLE IF NOT EXISTS gongxu_ship_stage(
 ship_no TEXT, ship_type TEXT, src_file TEXT, rev TEXT,
 total_hours REAL DEFAULT 0, group_json TEXT,
 added INTEGER DEFAULT 0, imported_at TEXT,
 UNIQUE(ship_no, src_file));
CREATE INDEX IF NOT EXISTS ix_sr_ver ON schedule_result(version_no);
CREATE INDEX IF NOT EXISTS ix_task_ship ON tasks(ship_no);
CREATE INDEX IF NOT EXISTS ix_tmpl_type ON task_templates(ship_type);
CREATE INDEX IF NOT EXISTS ix_dep ON task_dependencies(ship_type, process_code);
"""


def connect(db_path: str = DB_PATH):
    """
    打开数据库连接，带重试与友好报错。

    常见的 "unable to open database file" 原因：
      1. 同时开了多个本程序窗口，数据库被锁
      2. 程序放在 U 盘 / 移动硬盘 / 网盘同步文件夹（OneDrive 等）里运行，
         这些位置不支持 SQLite 的日志文件
      3. 安全软件正在拦截数据库文件的创建/写入
      4. 所在文件夹没有写入权限
    """
    import time
    last = None
    for _ in range(3):
        try:
            conn = sqlite3.connect(db_path, timeout=10)
            conn.row_factory = sqlite3.Row
            try:
                # WAL 模式更快，但 U 盘(FAT32)/网盘同步目录不支持，失败时退回默认模式
                conn.execute("PRAGMA journal_mode=WAL")
            except sqlite3.Error:
                pass
            conn.execute("PRAGMA busy_timeout=8000")
            return conn
        except sqlite3.Error as e:
            last = e
            time.sleep(0.4)
    raise RuntimeError(
        f"无法打开数据库文件：{db_path}\n原始错误：{last}\n\n请按顺序检查：\n"
        "  1. 是否同时打开了多个本程序窗口？请全部关闭，只重新打开一个\n"
        "  2. 程序是否放在 U 盘 / 移动硬盘 / 网盘同步文件夹里？"
        "请整个文件夹拷贝到电脑本地硬盘（例如 D:\\排产系统）后再运行\n"
        "  3. 安全软件（360、电脑管家等）是否弹出了拦截提示？请选择允许，"
        "并把本文件夹加入信任名单\n"
        "  4. 以上都不行时，把本文件夹拷贝到 D 盘根目录或桌面再试")


def _migrate_manpower(conn):
    """
    旧版 ship_manpower 带 UNIQUE(ship_no, team)，一条船一个班组只能有一段。
    新版要支持"分时间段设置不同占用人数"，需要去掉这个唯一约束。
    SQLite 不能直接删约束，这里做建新表 -> 迁移 -> 改名。
    """
    row = conn.execute(
        "SELECT sql FROM sqlite_master WHERE type='table' AND name='ship_manpower'").fetchone()
    if row is None:
        return
    if "UNIQUE" not in (row["sql"] or "").upper():
        return                       # 已经是新结构
    conn.executescript(
        "CREATE TABLE ship_manpower_new("
        " id INTEGER PRIMARY KEY AUTOINCREMENT, ship_no TEXT, team TEXT,"
        " workers INTEGER DEFAULT 0, new_workers INTEGER DEFAULT 0,"
        " start_date DATE, end_date DATE, note TEXT, seg INTEGER DEFAULT 0);"
        "INSERT INTO ship_manpower_new(ship_no,team,workers,new_workers,start_date,"
        " end_date,note,seg) SELECT ship_no,team,workers,new_workers,start_date,"
        " end_date,note,0 FROM ship_manpower;"
        "DROP TABLE ship_manpower;"
        "ALTER TABLE ship_manpower_new RENAME TO ship_manpower;")
    conn.commit()
    log("已升级人力占用表：支持分时间段设置占用人数")


def init_db(conn):
    conn.executescript(SCHEMA)
    _ensure_columns(conn)
    _migrate_manpower(conn)
    from precision_engine import migrate
    migrate(conn)
    # 班组（作息与加班默认值：周六不上班、周日不上班、每周加班 3 天 × 3 小时、新人不加班）
    wd = WORKTIME_DEFAULTS
    for i, code in enumerate(TEAM_CODES):
        c = DEFAULT_TEAMS[code]
        conn.execute(
            "INSERT OR IGNORE INTO teams(team_code,skilled,new_workers,sh_per_day,nh_per_day,"
            "work_sat,work_sun,overtime,ship_max,ship_max_u,max_per_task,sort_order,"
            "ot_days_per_week,ot_hours,new_overtime) VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)",
            (code, c["skilled"], c["new"], c["sh"], c["nh"], wd["work_sat"], wd["work_sun"],
             0.0, c["ship_max"], c["ship_max_u"], 4, i,
             wd["ot_days_per_week"], wd["ot_hours"], wd["new_overtime"]))
    for z, cap in DEFAULT_ZONE_CAPACITY.items():
        conn.execute("INSERT OR IGNORE INTO zones VALUES(?,?,?)", (z, z, cap))
    for a, b, sz, lv in DEFAULT_CONFLICTS:
        row = conn.execute("SELECT id FROM task_conflicts WHERE ta=? AND tb=?", (a, b)).fetchone()
        if not row:
            conn.execute("INSERT INTO task_conflicts(ta,tb,same_zone,scope,level,enabled) VALUES(?,?,?,?,?,?)",
                         (a, b, 1 if sz else 0, "zone" if sz else "ship", lv, 1 if lv == "禁止" else 0))
    conn.commit()


def _ensure_columns(conn):
    """轻量迁移：老数据库缺列时自动补齐（避免用户升级后表结构不一致）"""
    need = {"task_templates": [("milestone", "TEXT DEFAULT ''")],
            "actual_progress": [("option_flag", "TEXT")],
            "task_conflicts": [("scope", "TEXT DEFAULT 'ship'")],
            "ships": [("current_stage", "TEXT"),
                      ("manual_inspection", "DATE"),
                      ("fixed_workers", "INTEGER DEFAULT 0"),
                      ("fixed_new", "INTEGER DEFAULT 0"),
                      ("manpower_mode", "INTEGER DEFAULT 0"),
                      ("prediction_basis", "TEXT"),
                      ("feasibility_status", "TEXT"),
                      ("unscheduled_hours", "REAL DEFAULT 0")],
            "teams": [("ot_days_per_week", "INTEGER DEFAULT 3"),
                      ("ot_hours", "REAL DEFAULT 3.0"),
                      ("new_overtime", "INTEGER DEFAULT 0")]}
    for table, cols in need.items():
        have = {r[1] for r in conn.execute(f"PRAGMA table_info({table})")}
        for name, ddl in cols:
            if name not in have:
                conn.execute(f"ALTER TABLE {table} ADD COLUMN {name} {ddl}")
    conn.commit()


class IssueCollector:
    """数据质量问题收集器：任何单元格异常只记录，不中断导入"""

    def __init__(self, src=""):
        self.src = src
        self.items = []

    def add(self, sheet, row, col, issue, detail="", raw="", severity="警告"):
        if len(self.items) < 500:
            self.items.append((self.src, str(sheet), int(row or 0), str(col or ""),
                               issue, detail, str(raw)[:120], severity))

    def save(self, conn):
        if not self.items:
            return 0
        conn.executemany("INSERT INTO data_quality(src,sheet,row_no,col,issue,detail,raw,severity)"
                         " VALUES(?,?,?,?,?,?,?,?)", self.items)
        conn.commit()
        return len(self.items)

# ==========================================================================
#  模块：aps_import.py
# ==========================================================================
import datetime as dt
import os
import re
import shutil
from collections import defaultdict


_SHIP_SHEET_RE = re.compile(r"^[A-Za-z]+[0-9]{3,}$")

DESKTOP_CANDIDATES = {
    "master_plan": ["主计划执行情况表 2026-09-23.xlsx", "主计划执行情况表 2026-09-16.xlsx"],
    "team_plans": ["初装1组-总装1线.xlsx", "机管1组-总装1线.xlsx", "电装1组-总装1线.xlsx",
                   "甲板1组-总装1线.xlsx", "终装1组-总装1线.xlsx"],
    "templates": ["A36S_生产周期.xlsx", "A42C_生产周期.xlsx", "A45S007_生产周期_60生产日.xlsx"],
}


# ============================================================ 文件扫描
def scan_raw_files(raw_dir=RAW_DIR):
    r = {"master_plan": None, "team_plans": [], "templates": [], "realtime": [],
         "plan_detail": []}
    if not os.path.isdir(raw_dir):
        return r
    for fn in sorted(os.listdir(raw_dir)):
        if not fn.lower().endswith((".xlsx", ".xls")) or fn.startswith("~$"):
            continue
        p = os.path.join(raw_dir, fn)
        if "排布计划详情" in fn or "排布计划" in fn:
            r["plan_detail"].append(p)          # PMC 排布单（第三个交替验证源）
        elif "主计划" in fn:
            if r["master_plan"] is None or fn > os.path.basename(r["master_plan"]):
                r["master_plan"] = p
        elif "生产周期" in fn:
            r["templates"].append(p)
        elif "组" in fn and ("总装" in fn or "线" in fn):
            r["team_plans"].append(p)
        elif re.match(r"^[A-Za-z]+\d+", fn):     # A36S270.xlsx 这类单船实时表
            r["realtime"].append(p)
    return r


def norm_ship_no(v) -> str:
    """
    船号统一：**去掉所有空白字符（含空格 / 全角空格 / 制表符 / 换行）+ 转大写**。

    班组情况表里同一条船会出现 A28MC068 / A28mc068 两种写法，
    不统一就会把一条船拆成两条（工序和工时被分到两处，看着像"某个班组没了"）。

    2026-10-10 补充：以前只 strip() 首尾空格，**船号中间的空格原样保留**，
    于是「A46C 006」和「A46C006」被当成两条不同的船。用户在主计划里
    输错船号后没法覆盖，只能重导，结果库里两条并存 —— 剔除一条，
    另一条还在排产里冒出来。现在统一去掉内部空白，两者视为同一条船。
    """
    s = norm_text(v)
    if not s:
        return s
    # 去掉所有空白（普通空格 / 全角空格 \u3000 / 制表符 / 换行等），不区分大小写
    s = re.sub(r"\s+", "", s)
    s = s.replace("\u3000", "")
    return s.upper() if s else s


def merge_duplicate_ships(conn):
    """合并仅大小写不同的重复船号（例如 A28mc068 -> A28MC068）"""
    groups = defaultdict(list)
    for r in conn.execute("SELECT ship_no, plan_inspection FROM ships"):
        groups[r["ship_no"].upper()].append(r)
    merged = 0
    for up, rows in groups.items():
        if len(rows) < 2:
            continue
        # 主船号优先：本身就是大写写法的 -> 有交检日的
        rows.sort(key=lambda r: (0 if r["ship_no"] == up else 1,
                                 0 if r["plan_inspection"] else 1))
        main = rows[0]["ship_no"]
        for r in rows:
            v = r["ship_no"]
            if v == main:
                continue
            for tbl in ("tasks", "actual_progress", "realtime_progress"):
                try:
                    conn.execute(f"UPDATE OR IGNORE {tbl} SET ship_no=? WHERE ship_no=?",
                                 (main, v))
                    conn.execute(f"DELETE FROM {tbl} WHERE ship_no=?", (v,))
                except Exception as e:
                    log(f"合并船号 {v}->{main} 时 {tbl} 处理失败：{e}")
            conn.execute("DELETE FROM ships WHERE ship_no=?", (v,))
            merged += 1
            log(f"合并重复船号：{v} -> {main}")
    if merged:
        conn.commit()
        log(f"共合并 {merged} 个重复船号（大小写不一致）")
    return merged


def looks_like_ship_file(fn: str) -> bool:
    """
    文件名是否像"单船完成情况表"：A36S270.xlsx / A45S007 (1).xlsx / A70L007g.xlsx

    必须【整体】就是一个船号（允许尾部多 1~2 个字母数字，如 A70L007g）。
    不能只匹配前缀 —— 否则 "A36S_生产周期.xlsx" 这种【船型工艺模板】
    会被误当成单船完成情况表，把模板数据灌进进度表、把工时冲成 0。
    """
    base = os.path.splitext(os.path.basename(fn))[0]
    base = re.sub(r"\(\d+\)\s*$", "", base).strip()      # A45S007 (1) -> A45S007
    base = re.sub(r"[-_‐]\s*\d{6,}$", "", base).strip()  # 去掉尾部日期串
    # 含中文、下划线、空格的一律不是单船表
    if re.search(r"[\u4e00-\u9fff_\s]", base):
        return False
    # 整体必须是"字母数字"，且以"字母+数字"开头（A36S270 / A70L007g 都符合）
    return bool(re.match(r"^[A-Za-z]+[0-9]+[A-Za-z0-9]*$", base))


def find_realtime_dirs(desktop=None):
    """
    找到所有可能存放"单船实时进度表"的文件夹。

    之前只认名字里带 9-24/实时/进度 的文件夹，导致新导出的表（文件夹叫别的名字）
    读不到。现在放宽：只要有 .xlsx 且文件名像船号，就认为可能是。
    """
    if desktop is None:
        desktop = os.path.join(os.path.expanduser("~"), "Desktop")
    if not os.path.isdir(desktop):
        return []
    hits = []
    # 1) 桌面根目录本身就当成一个候选（很多用户直接把表丢在桌面）
    hits.append(desktop)
    # 2) 桌面下的子文件夹
    for name in sorted(os.listdir(desktop)):
        p = os.path.join(desktop, name)
        if not os.path.isdir(p) or name.startswith("."):
            continue
        try:
            has = any(looks_like_ship_file(f) and f.lower().endswith((".xlsx", ".xls"))
                      and not f.startswith("~$") for f in os.listdir(p))
        except OSError:
            continue
        if has:
            hits.append(p)
    # 名字明显相关的排前面
    def _score(p):
        n = os.path.basename(p)
        return (0 if any(k in n for k in ("9-24", "9.24", "实时", "进度", "报工", "拉出",
                                          "完成", "生产")) else 1)
    hits.sort(key=_score)
    return hits


def scan_realtime_files(conn=None, raw_dir=RAW_DIR):
    """
    汇总所有单船实时/完成情况表：
      * data/raw 里已复制的
      * settings 里手动指定的目录
      * 桌面上自动探测到的目录（含桌面根目录本身）
    """
    files = list(scan_raw_files(raw_dir)["realtime"])
    custom = None
    if conn is not None:
        r = conn.execute("SELECT value FROM settings WHERE key='realtime_dir'").fetchone()
        custom = (r["value"] if r else None) or None
    dirs = ([custom] if custom and os.path.isdir(custom) else []) + find_realtime_dirs()
    for d in dirs:
        try:
            for fn in sorted(os.listdir(d)):
                if not fn.lower().endswith((".xlsx", ".xls")) or fn.startswith("~$"):
                    continue
                if looks_like_ship_file(fn):
                    files.append(os.path.join(d, fn))
        except OSError:
            continue
    # 去重（同名文件只保留一个，优先 data/raw 与手动指定目录）
    seen, out = set(), []
    for p in files:
        k = os.path.basename(p).lower()
        if k in seen:
            continue
        seen.add(k)
        out.append(p)
    return out


def classify_excel(path) -> str:
    """判断一个 Excel 是哪类文件：
    实时进度表 / 工艺模板 / 班组周计划 / 主计划 / 工序工位 / 未知"""
    fn = os.path.basename(path)
    # 排布计划详情也含"工序编码+状态"，按内容会被误判成单船完成情况表，
    # 所以【先按文件名】判定，再走内容识别。
    if "排布计划" in fn:
        return "plan"
    try:
        wb = read_workbook(path)
    except Exception:
        return "unknown"
    for _name, rows in wb.items():
        for i in range(min(15, len(rows))):
            t = row_text(rows[i])
            if "工序编码" in t and "状态" in t:
                return "realtime"
            if "生产周期" in t or ("阶段" in t and "标准生产日" in t):
                return "template"
            if "船号" in t and ("班组" in t or "工艺" in t):
                return "unknown"
    # 「工序工位」文件：含「工时总表」工作表，且表头有 班组 + 合计（小时）
    if _GX_TOTAL_SHEET in wb:
        rows = wb[_GX_TOTAL_SHEET]
        for i in range(min(8, len(rows))):
            t = row_text(rows[i])
            if "班组" in t and "合计" in t:
                return "gongxu"
    for _name, rows in wb.items():
        for i in range(min(6, len(rows))):
            t = row_text(rows[i])
            if "班组" in t and "合计" in t:
                return "gongxu"
    if "排布计划" in fn:
        return "plan"
    if "生产周期" in fn:
        return "template"
    if "主计划" in fn:
        return "master"
    if "工序工位" in fn or "工时总表" in fn:
        return "gongxu"
    return "unknown"


def import_uploaded_file(conn, filename: str, data_b64: str, auto_import=True):
    """
    手动上传的 Excel：先保存到 data/raw，再按内容自动识别并导入。

    :return: {"file": 文件名, "kind": 类型, "result": 各类导入器的返回}
    """
    import base64
    try:
        raw = base64.b64decode(data_b64)
    except Exception as e:
        return {"file": filename, "error": f"文件解码失败：{e}"}
    if len(raw) > 40 * 1024 * 1024:
        return {"file": filename, "error": "文件超过 40MB"}

    safe = os.path.basename(filename)
    safe = re.sub(r'[\\/:*?"<>|]', "_", safe)
    if not safe.lower().endswith((".xlsx", ".xls")):
        safe += ".xlsx"
    os.makedirs(RAW_DIR, exist_ok=True)
    path = os.path.join(RAW_DIR, safe)
    with open(path, "wb") as f:
        f.write(raw)
    log(f"手动上传：{safe}（{len(raw)/1024:.0f} KB）")

    kind = classify_excel(path)
    out = {"file": safe, "path": path, "kind": kind, "bytes": len(raw)}
    if not auto_import:
        return out
    try:
        if kind == "realtime":
            out["result"] = import_realtime_progress(conn, path)
        elif kind == "plan":
            out["result"] = import_plan_detail(conn, path)
        elif kind == "template":
            t = import_cycle_template(conn)
            out["result"] = {"templates": len(t.get("types", [])) if isinstance(t, dict) else 0}
        elif kind == "gongxu":
            out["result"] = import_gongxu_hours(conn, path)
        else:
            out["result"] = {"skip": True, "hint": "未自动识别类型，可点『开始导入』让全量流程处理"}
    except Exception as e:
        out["error"] = f"导入失败：{e}"
    return out


def copy_from_desktop(desktop=None):
    """
    从桌面复制业务数据到 data/raw。

    用"模糊扫描"而不是死记文件名 —— 现实里文件名经常不规范
    （例如主计划文件被存成了《主计划执行情况表 2026-09-213.xlsx》）。
    """
    if desktop is None:
        desktop = os.path.join(os.path.expanduser("~"), "Desktop")
    if not os.path.isdir(desktop):
        return []
    copied = []

    def _grab(pred, limit=None, prefer_newest=False):
        """按条件挑出桌面上的文件并复制"""
        hits = []
        for fn in os.listdir(desktop):
            if not fn.lower().endswith((".xlsx", ".xls")) or fn.startswith("~$"):
                continue
            if pred(fn):
                hits.append(fn)
        if not hits:
            return []
        if prefer_newest:
            hits.sort(key=lambda f: os.path.getmtime(os.path.join(desktop, f)), reverse=True)
        if limit:
            hits = hits[:limit]
        out = []
        for fn in hits:
            src, dst = os.path.join(desktop, fn), os.path.join(RAW_DIR, fn)
            try:
                # 目标不存在，【或桌面版本更新】-> 覆盖复制。
                # 老逻辑是 `if not os.path.exists(dst)`：raw 里一旦有同名文件就永不更新，
                # 用户在桌面每周更新的班组周计划同步不进来，导入的始终是旧数据，
                # 于是"交叉验证工序完成情况"看起来失效（前端还报"已复制过"）。
                need = (not os.path.exists(dst)) or \
                       (os.path.getmtime(src) > os.path.getmtime(dst) + 1)
                if need:
                    shutil.copy2(src, dst)
                    out.append(fn)
            except Exception as e:
                log(f"复制失败 {fn}: {e}")
        return out

    # 主计划：只取最新的一份（避免多个历史版本混在一起）
    copied += _grab(lambda f: "主计划" in f, limit=1, prefer_newest=True)
    # 船型生产周期模板
    copied += _grab(lambda f: "生产周期" in f or "工位工序顺序表" in f)
    # 五班组周计划
    copied += _grab(lambda f: ("总装" in f or "组-" in f or "组" in f and "线" in f))
    # PMC 排布计划详情（第三个交替验证源）
    copied += _grab(lambda f: "排布计划" in f)
    # 工位工序文件（设计部下发的《xx 工序工位 REVx.xlsx》，2026-10-29 起也从桌面自动拉，
    # 之前要手动从共享文件夹复制 —— 用户要求"新文件可以从桌面读取"）
    copied += _grab(lambda f: "工序工位" in f or "工位工序" in f)
    return copied


# ============================================================ 船型工艺模板
def _pick_sheet(wb, keywords, exclude=("周计划", "物料", "统计")):
    names = list(wb.keys())
    for n in names:
        if any(x in n for x in exclude):
            continue
        if all(k in n for k in keywords):
            return n
    for n in names:
        if any(k in n for k in keywords):
            return n
    return None


# ============================================================ 工序工位文件（工时总表）
# 设计开发二部下发的《xxx 工序工位 REVx.xlsx》，其「工时总表」工作表结构：
#   第1行：公司名 + 年份（合并单元格）
#   第2行：列头 —— 船型 | 班组 | 铺糊-大件 | 铺糊-小件 | 泡沫 | 油漆 |
#                   表面处理-干磨 | 表面处理-水磨 | 脱模切割 | 初装 | 家装 | 终装 |
#                   甲板 | 不锈钢 | 机装 | 管装 | 电装 | [准备] | 合计（小时）
#   第3行起：每个船型一行，第2列固定为「总工时」。
#
# 注意：不同 REV 的文件列数不完全一样（有的多一列「准备」），
# 所以一律**按列头文字匹配**取数，绝不写死列号。
_GX_TOTAL_SHEET = "工时总表"

# 工时总表里的原始班组列（用于整体存档，不丢数据）
_GX_RAW_COLS = ["铺糊-大件", "铺糊-小件", "泡沫", "油漆", "表面处理-干磨",
                "表面处理-水磨", "脱模切割", "初装", "家装", "终装", "甲板",
                "不锈钢", "机装", "管装", "电装", "准备"]

# 五个作业组的口径（已与 PMC 确认）：
#   机管组 = 工时总表里的「机装」+「管装」两列相加，其余一一对应。
# 只影响写入 ship_group_hours 的五组工时；17 列原始明细照旧全量存进 hours_raw。
GX_GROUPS = [
    ("初装", ["初装"]),
    ("机管", ["机装", "管装"]),
    ("电装", ["电装"]),
    ("甲板", ["甲板"]),
    ("终装", ["终装"]),
]


def _gx_norm_col(s):
    """列头归一化：去掉换行/空格/全角括号差异，便于匹配"""
    return re.sub(r"[\s\u3000]", "", str(s or "")).strip()


def _to_float(v):
    try:
        if v is None or v == "":
            return None
        return float(str(v).replace(",", "").strip())
    except (TypeError, ValueError):
        return None


def _gx_ship_family(s):
    """
    取"船族"前缀：A36MC005 -> A36MC，A35S003 -> A35S，A28MC -> A28MC。
    用于判断工时总表里某一行是不是本文件的"对照行"。

    注意：船型列可能写成 "A36MC Rev1"（REV 是版本号，不是船族的一部分），
    所以先按 REV 切一刀，再取「字母 + 两位数字 + 至多 3 个字母」。
    """
    s = re.sub(r"[^A-Za-z0-9]", "", str(s or "")).upper()
    s = re.split(r"REV", s)[0] or s          # "A36MCREV1" -> "A36MC"
    m = re.match(r"^([A-Z]+\d{2,3}[A-Z]{0,3})", s)
    return m.group(1) if m else s


def _gx_same_family(a, b):
    """两个船族是否算同一族：相等，或前者是后者的前缀（A36MC 与 A36MCC）"""
    if not a or not b:
        return False
    return a == b or a.startswith(b) or b.startswith(a)


def _gx_is_current_ship(row_ship_type, row_label, file_ship, file_family):
    """
    判断工时总表里的这一行是不是**本文件对应那条船**的行。

    现实情况：设计部给一份《A36MC005 工序工位》，里面的「工时总表」往往
    还挂着一条 A28MC 的老行（打印时隐藏的对照标准），它不属于 A36MC005。
    如果照着文件名把两条都记成 A36MC005，就会污染数据。

    判定顺序：
      1) 列头文字里含本文件船号 -> 是本船行；
      2) 行的船型族与文件船族不一致 -> 不是本船行；
      3) 表内合计为 0 / 空 -> 不是有效行。
    """
    row_type = str(row_ship_type or "").strip()
    lbl = str(row_label or "")
    if file_ship and file_ship in lbl:
        return True
    fam = _gx_ship_family(row_type)
    if not fam:
        return False
    if file_family and not _gx_same_family(fam, file_family):
        return False
    return True


def parse_gongxu_file(path):
    """
    解析一个「工序工位」文件，返回工时总表里的每个船型行。

    :return: {
      "file": 文件名, "rev": 版本号,
      "rows": [{"ship_type": 船型, "ship_no": 从文件名/表内推断的船号,
                "raw": {原始列名: 工时}, "groups": {组名: 工时},
                "total": 表内合计, "calc_total": 逐列加总}],
      "raw_cols": 实际出现的原始列名列表,
      "errors": [...]
    }
    """
    fname = os.path.basename(path)
    out = {"file": fname, "rev": "", "rows": [], "raw_cols": [], "errors": []}
    m = re.search(r"REV\.?\s*([0-9]+(?:\.[0-9]+)?)", fname, re.I)
    if m:
        out["rev"] = "REV" + m.group(1)

    try:
        wb = read_workbook(path)
    except Exception as e:
        out["errors"].append(f"读取失败：{e}")
        return out

    sheet = _pick_sheet(wb, [_GX_TOTAL_SHEET])
    if sheet is None:
        # 退化：找任意含「合计」且含「班组」的表
        for n, rows in wb.items():
            for i in range(min(6, len(rows))):
                t = row_text(rows[i])
                if "合计" in t and "班组" in t:
                    sheet = n
                    break
            if sheet:
                break
    if sheet is None:
        out["errors"].append("未找到「工时总表」工作表")
        return out
    out["sheet"] = sheet

    rows = wb[sheet]
    # 定位列头行
    hi = -1
    for i in range(min(8, len(rows))):
        t = row_text(rows[i])
        if "班组" in t and ("合计" in t or "船型" in t):
            hi = i
            break
    if hi < 0:
        out["errors"].append("未找到工时总表表头行")
        return out
    headers = rows[hi]

    # 列头 → 列号
    norm2idx = {}
    for ci, cell in enumerate(headers):
        k = _gx_norm_col(cell)
        if k and k not in norm2idx:
            norm2idx[k] = ci

    def col(*names):
        for n in names:
            k = _gx_norm_col(n)
            if k in norm2idx:
                return norm2idx[k]
            # 模糊：允许「合计（小时）」「合计(小时)」「合计」
            for kk, ci in norm2idx.items():
                if k and (kk.startswith(k) or k.startswith(kk)):
                    return ci
        return -1

    c_type = col("船型")
    c_team = col("班组")
    c_total = col("合计（小时）", "合计")
    raw_idx = {}
    for rc in _GX_RAW_COLS:
        ci = col(rc)
        if ci >= 0:
            raw_idx[rc] = ci
    # 「铺糊-大件」列头在不同文件里可能是「铺糊 大件」等，做一次容错
    for rc, aliases in (("铺糊-大件", ["铺糊大件", "铺糊-大件"]),
                        ("铺糊-小件", ["铺糊小件", "铺糊-小件"]),
                        ("表面处理-干磨", ["表面处理干磨", "表面处理-干磨"]),
                        ("表面处理-水磨", ["表面处理水磨", "表面处理-水磨"])):
        if rc not in raw_idx:
            for a in aliases:
                ci = norm2idx.get(_gx_norm_col(a), -1)
                if ci >= 0:
                    raw_idx[rc] = ci
                    break
    out["raw_cols"] = list(raw_idx.keys())
    out["col_map"] = {"船型": c_type, "班组": c_team, "合计": c_total}

    if c_type < 0:
        out["errors"].append("表头里找不到「船型」列")
        return out

    # 从文件名兜底取船号（例如 A35S003 工序工位 REV0.xlsx）
    fn_ship = ""
    m = re.match(r"\s*([A-Za-z]{1,3}\d{2,4}[A-Za-z]*\d{0,4})", fname)
    if m:
        fn_ship = m.group(1).upper()
    fn_family = _gx_ship_family(fn_ship)

    for ri in range(hi + 1, len(rows)):
        r = rows[ri]
        if not r:
            continue
        ship_type = str(r[c_type] if c_type < len(r) else "").strip()
        if not ship_type:
            continue
        team_lbl = str(r[c_team] if (0 <= c_team < len(r)) else "").strip()
        # 只认「总工时」行
        if c_team >= 0 and team_lbl and "总工时" not in team_lbl:
            continue
        # 跳过页脚
        if any(x in ship_type for x in ("编制", "审核", "批准", "日期", "OA流水", "船型")):
            continue

        raw = {}
        for rc, ci in raw_idx.items():
            v = r[ci] if ci < len(r) else None
            fv = _to_float(v)
            if fv is not None:
                raw[rc] = fv
        if not raw:
            continue

        groups = {}
        for gname, cols in GX_GROUPS:
            s = 0.0
            hit = False
            for c in cols:
                if c in raw:
                    s += raw[c]
                    hit = True
            groups[gname] = round(s, 2) if hit else 0.0

        total = _to_float(r[c_total]) if (0 <= c_total < len(r)) else None
        calc = round(sum(raw.values()), 2)

        # 船号推断：文件名里的船号优先（REV 文件通常一文件一船）
        ship_no = fn_ship or ""
        if not ship_no:
            m2 = re.search(r"([A-Za-z]{1,3}\d{2,4}[A-Za-z]*\d{0,4})", ship_type)
            ship_no = m2.group(1).upper() if m2 else ""

        # 表内「A36MC005 定额工分」这种列头也能提供船号
        if not ship_no:
            for cell in rows[hi]:
                m3 = re.search(r"\b([A-Z]\d{2}[A-Z]{1,3}\d{3})\b", str(cell or ""))
                if m3:
                    ship_no = m3.group(1).upper()
                    break

        # 过滤"对照行"：本文件船号是 A36MC005 时，表里挂着的 A28MC 老行不算本船数据。
        # 只对"从文件名取到船号"的情况生效；否则（表内自带多船）保持原样全部保留。
        is_current = _gx_is_current_ship(ship_type, team_lbl, fn_ship, fn_family)
        if fn_ship and not is_current:
            out.setdefault("skipped_rows", []).append({
                "row_no": ri + 1, "ship_type": ship_type.split("\n")[0].strip(),
                "reason": f"船型族与本文件船号 {fn_ship} 不一致，判定为对照行"})
            continue

        out["rows"].append({
            "row_no": ri + 1,
            "ship_type": ship_type.split("\n")[0].strip(),
            "ship_no": ship_no,
            "group_label": team_lbl or "总工时",
            "raw": raw, "groups": groups,
            "total": total, "calc_total": calc,
        })

    if not out["rows"]:
        out["errors"].append("工时总表里没有解析到任何有效船型行")
    return out


def import_gongxu_hours(conn, path):
    """导入「工序工位」文件的工时总表，落库到 ship_group_hours / gongxu_ship_stage"""
    p = parse_gongxu_file(path)
    fname = p.get("file") or os.path.basename(path)
    now = dt.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    n_hours = n_stage = 0
    for r in p["rows"]:
        sn = r["ship_no"]
        if not sn:
            continue
        conn.execute(
            "INSERT INTO gongxu_ship_stage(ship_no,ship_type,src_file,rev,total_hours,"
            "group_json,added,imported_at) VALUES(?,?,?,?,?,?,"
            "COALESCE((SELECT added FROM gongxu_ship_stage WHERE ship_no=? AND src_file=?),0),?)"
            " ON CONFLICT(ship_no,src_file) DO UPDATE SET ship_type=excluded.ship_type,"
            "rev=excluded.rev,total_hours=excluded.total_hours,group_json=excluded.group_json",
            (sn, r["ship_type"], fname, p["rev"], (r["total"] if r["total"] is not None
                                                   else r["calc_total"]),
             json.dumps({"raw": r["raw"], "groups": r["groups"]}, ensure_ascii=False),
             sn, fname, now))
        n_stage += 1
        for gname, gv in r["groups"].items():
            conn.execute(
                "INSERT INTO ship_group_hours(ship_no,ship_type,group_code,hours,hours_raw,"
                "src_file,rev,updated_at) VALUES(?,?,?,?,?,?,?,?)"
                " ON CONFLICT(ship_no,group_code) DO UPDATE SET hours=excluded.hours,"
                "ship_type=excluded.ship_type,hours_raw=excluded.hours_raw,"
                "src_file=excluded.src_file,rev=excluded.rev,updated_at=excluded.updated_at",
                (sn, r["ship_type"], gname, gv,
                 json.dumps(r["raw"], ensure_ascii=False), fname, p["rev"], now))
            n_hours += 1
    conn.commit()
    log(f"[{fname}] 工序工位：船型行 {len(p['rows'])}，写入五组工时 {n_hours} 条，"
        f"待确认船号 {n_stage} 条")
    return {"file": fname, "rev": p["rev"], "rows": len(p["rows"]),
            "hours": n_hours, "stage": n_stage, "sheets": [p.get("sheet") or ""],
            "parsed": p, "errors": p["errors"]}


_TMPL_COLS = {
    "班组": ["班组"], "工序编码": ["工序编码", "工序代码", "工序编号"],
    "工段": ["工段原值", "工段"], "工序名称": ["工序名称"],
    "作业区域": ["作业区域", "作业位置"], "工时": ["定额工时", "定额人时", "工时"],
    "开始日": ["计划开始日", "开始生产日", "开始日"],
    "结束日": ["计划结束日", "结束生产日", "结束日"],
    "物料": ["重点物料", "重点物件"], "前置": ["前置", "放行条件"],
}


def _map_cols(headers, spec):
    out = {}
    for field, kws in spec.items():
        for kw in kws:
            i = find_col(headers, kw)
            if i >= 0:
                out[field] = i
                break
    return out


def import_cycle_template(conn, path, ship_type=""):
    """导入一个船型的标准生产周期模板"""
    iss = IssueCollector(os.path.basename(path))
    fname = os.path.basename(path)
    wb = read_workbook(path)

    if not ship_type:
        m = re.match(r"^([A-Za-z]+\d*[A-Za-z]*)", os.path.splitext(fname)[0])
        if m:
            tok = m.group(1).upper()
            m2 = re.match(r"^([A-Z]+\d*[A-Z]*?)\d{2,}$", tok)
            ship_type = m2.group(1) if m2 else tok
    if not ship_type:
        for n in wb.keys():
            m = re.search(r"([A-Za-z]+\d*[A-Za-z]*)\s", n or "")
            if m:
                t = resolve_ship_type(m.group(1))
                if t:
                    ship_type = t
                    break
    ship_type = (ship_type or os.path.splitext(fname)[0][:16]).upper()
    log(f"导入船型模板 {fname} -> {ship_type}")

    sheet = _pick_sheet(wb, ["工序明细"]) or _pick_sheet(wb, ["工序"])
    if sheet is None:
        iss.add("", 0, "", "未找到工序明细工作表", fname, "", "错误")
        iss.save(conn)
        return {"ship_type": ship_type, "processes": 0}

    rows = wb[sheet]
    hi = detect_header(rows, ["班组", "工序名称"], scan=12)
    if hi < 0:
        iss.add(sheet, 0, "", "未找到工序明细表头", "", "", "错误")
        iss.save(conn)
        return {"ship_type": ship_type, "processes": 0}
    cm = _map_cols(rows[hi], _TMPL_COLS)
    if "班组" not in cm or "工序编码" not in cm:
        iss.add(sheet, hi + 1, "", f"表头缺少关键列，识别到 {cm}", "", "", "错误")
        iss.save(conn)
        return {"ship_type": ship_type, "processes": 0}

    templates, seen = [], set()
    for ri in range(hi + 1, len(rows)):
        r = rows[ri]
        team = normalize_team(cell(r, cm.get("班组")))
        if not team:
            continue
        code = norm_text(cell(r, cm.get("工序编码")))
        if not code or code in seen:
            continue
        seen.add(code)
        name = norm_text(cell(r, cm.get("工序名称")))
        hours = to_number(cell(r, cm.get("工时")), 0.0)
        if hours <= 0:
            iss.add(sheet, ri + 1, "定额工时", "工时为 0", code, "", "提示")
        sd = parse_day_no(cell(r, cm.get("开始日")), 1)
        ed = max(sd, parse_day_no(cell(r, cm.get("结束日")), sd))
        templates.append((ship_type, code, name, team, norm_text(cell(r, cm.get("工段"))),
                          hours, sd, ed, norm_text(cell(r, cm.get("作业区域"))),
                          guess_task_type(name), norm_text(cell(r, cm.get("前置")))[:400],
                          norm_text(cell(r, cm.get("物料")))[:200]))

    code_set = {t[1] for t in templates}
    # ---- 依赖：显式前置 + 同班组按生产日串联 ----
    deps, dep_seen = [], set()
    proc_re = re.compile(r"[A-Za-z]{2,}[0-9]*[A-Za-z]*[0-9]{2,}")
    for t in templates:
        for m in proc_re.finditer(t[10]):
            pre = m.group(0)
            if pre != t[1] and pre in code_set and (pre, t[1]) not in dep_seen:
                dep_seen.add((pre, t[1]))
                deps.append((ship_type, t[1], pre, 0, "模板"))
    by_team = defaultdict(list)
    for t in templates:
        by_team[t[3]].append(t)
    for team, items in by_team.items():
        items.sort(key=lambda x: (x[6], x[7], x[1]))
        for a, b in zip(items, items[1:]):
            if b[6] <= a[6]:
                continue                     # 同一天开始 -> 允许并行
            if (a[1], b[1]) not in dep_seen:
                dep_seen.add((a[1], b[1]))
                deps.append((ship_type, b[1], a[1], 0, "自动生成"))

    # ---- 里程碑 ----
    miles = []
    cal_sheet = _pick_sheet(wb, ["每日生产周期"])
    if cal_sheet:
        mr = wb[cal_sheet]
        mile_row = day_row = -1
        for i in range(min(40, len(mr))):
            t = row_text(mr[i])
            if "里程碑" in t or "放行" in t:
                mile_row = i
            if "生产日" in t:
                day_row = i
        if mile_row >= 0:
            for ci, v in enumerate(mr[mile_row]):
                txt = norm_text(v)
                if not txt:
                    continue
                m = re.search(r"\[?\bG(\d)\]?", txt)
                code_g = "G" + m.group(1) if m else f"M{ci}"
                day = parse_day_no(mr[day_row][ci], 0) if day_row >= 0 and ci < len(mr[day_row]) else 0
                miles.append((ship_type, code_g, txt[:120], day, 1 if m else 0))

    # ---- 用「每日生产周期」校正每道工序的生产日（核心）----
    # 「工序明细」只列关键工序，且 earliest_day 常年缺失（全库 85% 是 1）；
    # 而「每日生产周期」才是 PMC 真正排的顺序：每一行是一个班组，每一列是一个生产日，
    # 格子里写着那天要做的工序编码。同一道工序出现在多天 -> 取首日为 earliest、末日为 latest。
    if cal_sheet:
        mr = wb[cal_sheet]
        d_row = -1
        for i in range(min(60, len(mr))):
            if "生产日" in row_text(mr[i]):
                d_row = i
                break
        col_day = {}
        if d_row >= 0:
            for ci, v in enumerate(mr[d_row]):
                _d = parse_day_no(v, 0)
                if _d:
                    col_day[ci] = _d
        if col_day:
            _PROC = re.compile(r"[A-Z]\d{2}[A-Z]{1,2}\d{3,5}")
            dmap = {}                      # code -> [min_day, max_day, team]
            for i, row in enumerate(mr):
                if not row:
                    continue
                lab = norm_text(row[0])
                if not lab.endswith("工序"):
                    continue
                team = ""
                for _tk in ("初装", "机管", "电装", "甲板", "终装", "前置"):
                    if _tk in lab:
                        team = _tk
                        break
                for ci, day in col_day.items():
                    if ci >= len(row):
                        continue
                    for code in _PROC.findall(str(row[ci] or "")):
                        e = dmap.get(code)
                        if e is None:
                            dmap[code] = [day, day, team]
                        else:
                            e[0] = min(e[0], day)
                            e[1] = max(e[1], day)
            # 覆盖模板里已有工序的生产日
            idx = {t[1]: k for k, t in enumerate(templates)}
            hit = 0
            for code, (d0, d1, tm_) in dmap.items():
                k = idx.get(code)
                if k is None:
                    continue
                t = list(templates[k])
                t[6], t[7] = d0, d1
                if tm_ and not t[3]:
                    t[3] = tm_
                templates[k] = tuple(t)
                hit += 1
            log("每日生产周期：解析出 %d 道工序的生产日，覆盖模板 %d 道（%s 共 %d 个生产日）"
                % (len(dmap), hit, ship_type, max(col_day.values())))
            _miss = [c for c in dmap if c not in idx]
            if _miss:
                log("  其中 %d 道工序在「工序明细」里没有，仅生产周期表有：%s%s"
                    % (len(_miss), "、".join(_miss[:5]), "…" if len(_miss) > 5 else ""))

    # ---- 落库（覆盖式）----
    # 特殊组名映射（如"不锈钢"→"甲板"，2026-10-29 PMC 确认的口径）
    if TEAM_REMAP:
        templates = [tuple(TEAM_REMAP.get(t[3], t[3]) if i == 3 else v
                           for i, v in enumerate(t)) for t in templates]
    conn.execute("DELETE FROM task_templates WHERE ship_type=?", (ship_type,))
    conn.execute("DELETE FROM task_dependencies WHERE ship_type=?", (ship_type,))
    conn.execute("DELETE FROM milestones WHERE ship_type=?", (ship_type,))
    conn.executemany(
        "INSERT INTO task_templates(ship_type,process_code,process_name,team,section,"
        "standard_hours,earliest_day,latest_day,zone,task_type,predecessor_text,material_text)"
        " VALUES(?,?,?,?,?,?,?,?,?,?,?,?)", templates)
    if deps:
        conn.executemany("INSERT INTO task_dependencies(ship_type,process_code,predecessor_code,"
                         "lag_days,source) VALUES(?,?,?,?,?)", deps)
    if miles:
        conn.executemany("INSERT INTO milestones(ship_type,code,name,plan_day,is_gate)"
                         " VALUES(?,?,?,?,?)", miles)
    total_h = round(sum(t[5] for t in templates), 1)
    cycle = max((t[7] for t in templates), default=0)
    conn.execute("INSERT OR REPLACE INTO ship_types(type_code,cycle_days,total_hours,"
                 "process_count,source_file) VALUES(?,?,?,?,?)",
                 (ship_type, cycle, total_h, len(templates), fname))
    # 新出现的区域自动登记
    known = {r[0] for r in conn.execute("SELECT zone_code FROM zones")}
    new_zones = []
    for t in templates:
        z = t[8]
        if z and z not in known:
            known.add(z)
            cap = 2
            for k, v in DEFAULT_ZONE_CAPACITY.items():
                if k in z:
                    cap = v
                    break
            new_zones.append((z, z, cap))
    if new_zones:
        conn.executemany("INSERT OR IGNORE INTO zones VALUES(?,?,?)", new_zones)
    conn.commit()
    iss.save(conn)
    log(f"船型 {ship_type}: 工序 {len(templates)} 道 / 依赖 {len(deps)} 条 / "
        f"里程碑 {len(miles)} / 总工时 {total_h}H / 周期 {cycle} 生产日")
    return {"ship_type": ship_type, "processes": len(templates), "deps": len(deps),
            "milestones": len(miles), "hours": total_h, "cycle_days": cycle}


# ============================================================ 主计划
_NODE_FIELDS = [("船底启动", "plan_start"), ("船底脱模", "plan_demould"),
                ("甲板粘结", "plan_deck_bond"), ("甲板合拢", "plan_deck_bond"),
                ("飞桥粘结", "plan_flybridge"), ("飞桥合拢", "plan_flybridge"),
                ("硬顶", "plan_hardtop"), ("验船师首检", "plan_inspection"),
                ("质量部首检", "plan_inspection"), ("华鹰首检", "plan_inspection"),
                ("交检", "plan_inspection")]


def _import_master_sheet(conn, wb, sheet, rows, hi, iss, fname):
    headers = rows[hi]
    col_hull = find_col(headers, "Hull")
    col_hull = col_hull if col_hull >= 0 else 0
    col_dest = find_col(headers, "目的地")
    cols = {
        "sale_start": find_col(headers, "船底启动", 1),
        "sale_demould": find_col(headers, "船底脱模", 1),
        "plan_start": find_col(headers, "船底启动", 2),
        "plan_demould": find_col(headers, "船底脱模", 2),
        "plan_deck_bond": find_col(headers, "甲板粘结", 1) if find_col(headers, "甲板粘结", 1) >= 0
        else find_col(headers, "甲板合拢", 2),
        "plan_hardtop": find_col(headers, "硬顶", 1),
        "plan_inspection": find_col(headers, "交检", 1),
    }
    col_remark = -1
    for kw in ("预警", "备注", "异常", "风险"):
        i = find_col(headers, kw)
        if i >= 0:
            col_remark = i
            break

    today = dt.date.today()
    added = updated = skipped = 0
    blocked = _excluded_set(conn)          # 界面上叉掉的船：不许复活
    for ri in range(hi + 1, len(rows)):
        r = rows[ri]
        ship_no = norm_ship_no(cell(r, col_hull))
        if not ship_no or ship_no.lower() in {"hull#", "船号", "合计"}:
            continue
        if ship_no.strip().upper() in blocked:
            skipped += 1
            continue
        insp = to_date(cell(r, cols["plan_inspection"]))
        start = to_date(cell(r, cols["plan_start"]))
        if insp is None and start is None:
            skipped += 1
            continue
        if insp and insp < today - dt.timedelta(days=30):
            status = "完工"
        elif insp and insp > today + dt.timedelta(days=120):
            status = "未开工"
        else:
            status = "在制"
        vals = dict(
            ship_no=ship_no, ship_type=resolve_ship_type(ship_no),
            destination=norm_text(cell(r, col_dest)),
            sale_start=to_date(cell(r, cols["sale_start"])),
            sale_demould=to_date(cell(r, cols["sale_demould"])),
            plan_start=start, plan_demould=to_date(cell(r, cols["plan_demould"])),
            plan_deck_bond=to_date(cell(r, cols["plan_deck_bond"])),
            plan_hardtop=to_date(cell(r, cols["plan_hardtop"])),
            plan_inspection=insp, status=status,
            remarks=norm_text(cell(r, col_remark)) if col_remark >= 0 else "")
        # 同号覆盖：匹配时忽略大小写和船号里的空格，并优先删掉历史脏行。
        # 否则库里一条 "A46C 006"、Excel 里写 "A46C006"，会被当成两条船各存一份，
        # 用户改对了 Excel 重导也覆盖不掉，看着就是"这条船删了还会再出现"。
        row, dup_removed = _find_ship_row_fuzzy(conn, ship_no)
        if row is None:
            conn.execute(
                "INSERT INTO ships(ship_no,ship_type,destination,sale_start,sale_demould,"
                "plan_start,plan_demould,plan_deck_bond,plan_hardtop,plan_inspection,status,remarks)"
                " VALUES(:ship_no,:ship_type,:destination,:sale_start,:sale_demould,:plan_start,"
                ":plan_demould,:plan_deck_bond,:plan_hardtop,:plan_inspection,:status,:remarks)", vals)
            added += 1
        else:
            real = row["ship_no"]
            # 库里写法跟文件不一致（多空格 / 大小写不同）-> 顺手把船号规整成文件里的写法
            if real != ship_no:
                _rename_ship(conn, real, ship_no)
                if dup_removed:
                    log(f"主计划导入：{fname} 中 {ship_no} 覆盖了脏船号行 "
                        f"{'、'.join(dup_removed)}")
            sets = ", ".join(f"{k}=:{k}" for k, v in vals.items() if k != "ship_no" and v not in (None, ""))
            if sets:
                conn.execute(f"UPDATE ships SET {sets} WHERE ship_no=:ship_no", vals)
            updated += 1
    conn.commit()
    return {"ships": added, "updated": updated, "skipped": skipped}


def _find_ship_row_fuzzy(conn, ship_no):
    """
    按"去空格 + 大写"匹配库里的船行，返回 (row, 被清掉的脏船号列表)。

    匹配规则：完全相同的优先；没有就找"去掉空格后相同"的那条。
    无论哪种情况，**都会顺带扫一遍是否还有别的脏写法**（例如同时存在
    "A46C006" 和 "A46C 006"），把多余的连工序一起清掉 —— 否则用户改对了
    Excel 重导，脏的那条还在，就是"删了还会再出现"的根源。
    """
    key = norm_ship_no(ship_no)
    norm_map = {}
    for r in conn.execute("SELECT * FROM ships"):
        if norm_ship_no(r["ship_no"]) == key:
            norm_map.setdefault(r["ship_no"], r)
    if not norm_map:
        return None, []

    if ship_no in norm_map:
        keep_no = ship_no
    else:
        # 优先留"写法干净"的那条（没有空格），否则留最短的
        keep_no = sorted(norm_map.keys(),
                         key=lambda s: (bool(re.search(r"\s", s)), len(s)))[0]

    removed = []
    for no in list(norm_map.keys()):
        if no == keep_no:
            continue
        try:
            delete_ship(conn, no, with_tasks=True)
            removed.append(no)
        except Exception as e:
            log(f"清理重复船号 {no} 失败：{e}")
    if removed:
        conn.commit()
    row = conn.execute("SELECT * FROM ships WHERE ship_no=?", (keep_no,)).fetchone()
    return row, removed


def _rename_ship(conn, old_no, new_no):
    """把一条船的船号改名，连同它的所有从属数据一起改过去"""
    if old_no == new_no:
        return
    conn.execute("UPDATE ships SET ship_no=? WHERE ship_no=?", (new_no, old_no))
    for tbl, col in _SHIP_CHILD_TABLES:
        try:
            conn.execute("UPDATE %s SET %s=? WHERE %s=?" % (tbl, col, col),
                         (new_no, old_no))
        except Exception:
            pass


def _import_ship_progress_sheets(conn, wb, iss, fname):
    """解析主计划文件里每个船号的独立进度 sheet（A36082 等）"""
    added = updated = 0
    today = dt.date.today()
    for name, rows in wb.items():
        if not _SHIP_SHEET_RE.match((name or "").strip()):
            continue
        hi = -1
        for i in range(min(40, len(rows))):
            t = row_text(rows[i])
            if "船底启动" in t or "项目/日期" in t:
                hi = i
                break
        if hi < 0:
            continue
        headers = rows[hi]
        cand = defaultdict(list)
        for kw, field in _NODE_FIELDS:
            i = find_col(headers, kw)
            if i >= 0 and i not in cand[field]:
                cand[field].append(i)
        plan_row = None
        for r in rows[hi + 1:hi + 4]:
            if not row_text(r):
                continue
            if "实际" in row_text(r):
                continue
            plan_row = r
            break
        if plan_row is None:
            continue
        vals = {}
        for field, idxs in cand.items():
            for i in idxs:
                v = to_date(cell(plan_row, i))
                if v:
                    vals[field] = v
                    break
        if not vals:
            continue
        ship_no = norm_ship_no(name)
        if str(ship_no).strip().upper() in _excluded_set(conn):
            continue                       # 界面上已作废的船，单船 sheet 也不许复活
        insp = vals.get("plan_inspection")
        if insp and insp < today - dt.timedelta(days=30):
            status = "完工"
        elif insp and insp > today + dt.timedelta(days=120):
            status = "未开工"
        else:
            status = "在制"
        d = dict(vals)
        d.update(ship_no=ship_no, ship_type=resolve_ship_type(ship_no), status=status)
        # 同号覆盖：忽略大小写和空格，脏船号行顺手清掉（口径同主表）
        row, dup_removed = _find_ship_row_fuzzy(conn, ship_no)
        keys = ["ship_type", "plan_start", "plan_demould", "plan_deck_bond", "plan_hardtop",
                "plan_inspection", "status"]
        if row is None:
            conn.execute("INSERT INTO ships(ship_no,ship_type,plan_start,plan_demould,"
                         "plan_deck_bond,plan_hardtop,plan_inspection,status)"
                         " VALUES(:ship_no,:ship_type,:plan_start,:plan_demould,:plan_deck_bond,"
                         ":plan_hardtop,:plan_inspection,:status)",
                         {k: d.get(k) for k in ["ship_no", "ship_type"] + keys[1:]})
            added += 1
        else:
            real = row["ship_no"]
            if real != ship_no:
                _rename_ship(conn, real, ship_no)
                if dup_removed:
                    log(f"主计划导入(单船sheet)：{ship_no} 覆盖了脏船号行 "
                        f"{'、'.join(dup_removed)}")
            sets = ", ".join(f"{k}=:{k}" for k in keys if d.get(k) not in (None, ""))
            if sets:
                conn.execute(f"UPDATE ships SET {sets} WHERE ship_no=:ship_no",
                             {k: d.get(k) for k in keys + ["ship_no"]})
            updated += 1
    conn.commit()
    return {"ships": added, "updated": updated}


def import_master_plan(conn, path):
    iss = IssueCollector(os.path.basename(path))
    fname = os.path.basename(path)
    wb = read_workbook(path)
    total = {"ships": 0, "updated": 0, "skipped": 0, "sheets": []}
    for name, rows in wb.items():
        if _SHIP_SHEET_RE.match((name or "").strip()):
            continue
        if not rows:
            continue
        hi = -1
        for i in range(min(10, len(rows))):
            t = row_text(rows[i])
            if "交检" in t or ("船底启动" in t and "Hull" not in t):
                hi = i
                break
        if hi < 0:
            continue
        near = "".join(row_text(rows[i]) for i in range(max(0, hi - 2), min(len(rows), hi + 1)))
        if "Hull" not in near and "船号" not in near:
            continue
        r = _import_master_sheet(conn, wb, name, rows, hi, iss, fname)
        total["ships"] += r["ships"]
        total["updated"] += r["updated"]
        total["skipped"] += r["skipped"]
        total["sheets"].append({"sheet": name, **r})
    sp = _import_ship_progress_sheets(conn, wb, iss, fname)
    total["ship_sheets"] = sp
    iss.save(conn)
    log(f"主计划导入：新增 {total['ships']}，更新 {total['updated']}，"
        f"单船 sheet 新增 {sp['ships']} 更新 {sp['updated']}")
    return total


# ============================================================ 五班组周计划
_TEAM_COLS = {
    "船号": ["船号"], "工序编号": ["工序编号", "工序编码", "工序代码"],
    "工序名称": ["工序名称"], "班组": ["班组"], "工段": ["工段"],
    "工时": ["工时", "定额工时", "定额人时"],
    "需求完成时间": ["需求完成时间"], "周计划完成时间": ["周计划完成时间"],
    "实际完成时间": ["实际完成时间"], "状态": ["状态"],
    "未作业原因": ["未作业原因"], "备注": ["备注"],
    "选配项": ["选配项", "选配", "选项", "标配/选配"],
}

# 选配项取这些值时，该工序不做时间统计（不计工时、不参与排产）
OPTION_SKIP_VALUES = {"o", "0", "○", "〇", "〇"}


def _pick_main_sheet(wb, team_code):
    best, best_len = None, -1
    for name, rows in wb.items():
        if team_code and team_code in name and "周" not in name:
            if len(rows) > best_len:
                best, best_len = name, len(rows)
    if best:
        return best
    for name, rows in wb.items():                 # 退化：取行数最多
        if len(rows) > best_len:
            best, best_len = name, len(rows)
    return best


def import_team_plan(conn, path, team_code=""):
    iss = IssueCollector(os.path.basename(path))
    fname = os.path.basename(path)
    if not team_code:
        team_code = normalize_team(fname.split("-")[0])
    wb = read_workbook(path)
    sheet = _pick_main_sheet(wb, team_code)
    if sheet is None:
        return {"team": team_code, "records": 0}
    rows = wb[sheet]
    hi = detect_header(rows, ["船号", "工序"], scan=15)
    if hi < 0:
        iss.add(sheet, 0, "", "未找到班组计划表头", "", "", "错误")
        iss.save(conn)
        return {"team": team_code, "records": 0}
    cm = _map_cols(rows[hi], _TEAM_COLS)
    if "船号" not in cm or ("工序编号" not in cm and "工序名称" not in cm):
        iss.add(sheet, hi + 1, "", f"表头缺少关键列 {cm}", "", "", "错误")
        iss.save(conn)
        return {"team": team_code, "records": 0}
    log(f"[{fname}] sheet={sheet} 表头行={hi} 列映射={cm}")

    recs, seen = [], set()
    for ri in range(hi + 1, len(rows)):
        r = rows[ri]
        if is_blank_row(r):
            continue
        ship_no = norm_ship_no(cell(r, cm.get("船号")))
        if not ship_no:
            continue
        code = norm_text(cell(r, cm.get("工序编号")))
        name = norm_text(cell(r, cm.get("工序名称")))
        if not code:
            if not name:
                continue
            code = "NAME_" + str(abs(hash(name)) % 100000)
        key = (ship_no, code)
        if key in seen:
            continue
        seen.add(key)
        raw_status = norm_text(cell(r, cm.get("状态")))
        status = STATUS_ALIAS.get(raw_status, STATUS_TODO)
        for k, v in STATUS_ALIAS.items():
            if k in raw_status:
                status = v
                break
        hours = to_number(cell(r, cm.get("工时")), 0.0)
        if hours <= 0 and cm.get("工时") is not None:
            iss.add(sheet, ri + 1, "工时", "工时为 0 或无法解析", cell(r, cm.get("工时")), "", "提示")
        actual = to_date(cell(r, cm.get("实际完成时间")))
        if actual and status == STATUS_TODO:
            status = STATUS_DONE
        opt = norm_text(cell(r, cm.get("选配项")))
        recs.append((normalize_team(cell(r, cm.get("班组"))) or team_code, ship_no, code, name,
                     norm_text(cell(r, cm.get("工段"))), hours,
                     to_date(cell(r, cm.get("周计划完成时间"))), actual, raw_status, status,
                     norm_text(cell(r, cm.get("未作业原因"))),
                     1.0 if actual else (0.95 if status == STATUS_DONE else 0.6),
                     fname, sheet, opt))

    conn.execute("DELETE FROM actual_progress WHERE team=? AND source_file=?", (team_code, fname))
    conn.executemany("INSERT INTO actual_progress(team,ship_no,process_code,process_name,section,"
                     "hours,week_finish,actual_finish,raw_status,status,not_done_reason,confidence,"
                     "source_file,source_sheet,option_flag) VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)",
                     recs)
    conn.commit()
    ships = {r[1] for r in recs}
    iss.save(conn)
    n_opt = sum(1 for x in recs if str(x[-1] or "").strip().lower() in OPTION_SKIP_VALUES)
    log(f"[{fname}] 班组 {team_code} 导入 {len(recs)} 条，涉及 {len(ships)} 条船"
        + (f"，其中选配(O/0) {n_opt} 道不做时间统计" if n_opt else ""))
    return {"team": team_code, "records": len(recs), "ships": len(ships), "option_skipped": n_opt}


# ============================================================ 单船实时进度表（9-24 这类）
_RT_COLS = {
    "船号": ["船号"], "工段": ["工段"], "工序编码": ["工序编码", "工序代码", "工序编号"],
    "工序名称": ["工序名称"], "班组": ["班组"], "定额工时": ["定额工时"],
    "开工时间": ["开工时间"], "完工时间": ["完工时间"],
    "剩余工时": ["剩余工时"], "状态": ["状态"],
}


def _ship_no_from_filename(fn: str) -> str:
    """从文件名推断船号：A36S270.xlsx -> A36S270；A45S007 (1).xlsx -> A45S007"""
    base = os.path.splitext(os.path.basename(fn))[0]
    base = re.sub(r"\(\d+\)\s*$", "", base).strip()          # A45S007 (1) -> A45S007
    # A70L007g -> A70L007（同一条船被另存时尾部多带一个字母）
    m = re.match(r"^([A-Za-z]+[0-9]+)[A-Za-z]?$", base)
    if m:
        return m.group(1).upper()
    m = re.match(r"^([A-Za-z]+[0-9]+[A-Za-z0-9]*)", base)
    return (m.group(1) if m else base).upper()


def import_realtime_progress(conn, path) -> dict:
    """
    导入"单船实时进度表"（如桌面 9-24 文件夹里的 A36S270.xlsx）。

    表格特征：
      * 标题在第 1~2 行（如 " A36S270查询报工"），表头通常在第 6 行
      * 关键列：船号 / 工段 / 工序编码 / 工序名称 / 班组 / 定额工时(m) /
               开工时间 / 完工时间 / 剩余工时(m) / 状态
      * **工时单位是分钟**（720 分钟 = 12 小时），需要换算成小时
      * 状态：已完工 / 加工中 / 未开始（空格按"已完成"处理）
    """
    fname = os.path.basename(path)
    iss = IssueCollector(fname)
    wb = read_workbook(path)

    # 找表头行：必须同时有"工序编码"和"状态"
    sheet, hi, cm = None, -1, {}
    for name, rows in wb.items():
        for i in range(min(15, len(rows))):
            t = row_text(rows[i])
            if "工序编码" in t and "状态" in t:
                c = _map_cols(rows[i], _RT_COLS)
                if "工序编码" in c and "状态" in c:
                    sheet, hi, cm = name, i, c
                    break
        if hi >= 0:
            break
    if hi < 0:
        iss.add("", 0, "", "未找到实时进度表表头（需含工序编码+状态）", fname, "", "错误")
        iss.save(conn)
        return {"file": fname, "records": 0, "error": "表头不匹配"}

    rows = wb[sheet]
    default_ship = _ship_no_from_filename(fname)
    recs, seen = [], set()
    for ri in range(hi + 1, len(rows)):
        r = rows[ri]
        code = norm_text(cell(r, cm.get("工序编码")))
        if not code:
            continue
        ship_no = norm_ship_no(cell(r, cm.get("船号"))) or default_ship
        if not ship_no:
            continue
        key = (ship_no, code)
        if key in seen:
            continue
        seen.add(key)
        raw_status = norm_text(cell(r, cm.get("状态")))
        # 规则：已完工 / 加工中 / 空格 都算已完成，且不再计剩余工时
        done = 1 if raw_status in DONE_STATUS_RT else 0
        quota_min = to_number(cell(r, cm.get("定额工时")), 0.0)     # 分钟
        remain_min = to_number(cell(r, cm.get("剩余工时")), 0.0)    # 分钟
        quota_h = round(quota_min / 60.0, 2)
        remain_h = 0.0 if done else round(
            (remain_min / 60.0) if remain_min > 0 else quota_h, 2)
        recs.append((ship_no, code, norm_text(cell(r, cm.get("工序名称"))),
                     normalize_team(cell(r, cm.get("班组"))),
                     norm_text(cell(r, cm.get("工段"))),
                     quota_h, remain_h,
                     norm_text(cell(r, cm.get("开工时间")))[:30],
                     norm_text(cell(r, cm.get("完工时间")))[:30],
                     raw_status, done, fname))

    if recs:
        conn.executemany("INSERT OR REPLACE INTO realtime_progress"
                         "(ship_no,process_code,process_name,team,section,quota_hours,"
                         "remain_hours,start_time,finish_time,raw_status,done,source_file)"
                         " VALUES(?,?,?,?,?,?,?,?,?,?,?,?)", recs)
        conn.commit()
    iss.save(conn)
    log(f"[{fname}] 实时进度：{default_ship} 导入 {len(recs)} 道，"
        f"完成 {sum(1 for x in recs if x[10])} 道")
    return {"file": fname, "ship_no": default_ship, "records": len(recs),
            "done": sum(1 for x in recs if x[11])}


def import_plan_detail(conn, path) -> dict:
    """
    导入「排布计划详情」（PMC 排布单导出）。

    文件特征：前几行是查询时间 / 排布班组 / 排布单号 / 可出勤工时，
    表头通常在第 8 行左右，含：船号、工序编码、工序名称、所属班组、
    定额工时（H）、剩余工时（H）、状态。

    状态为 已完成/已完工/加工中/作业中 这类 -> 视为该工序完工，不计入工时。
    与班组情况表、单船工序完成情况表**交替验证（取并集）**。
    """
    fname = os.path.basename(path)
    iss = IssueCollector(fname)
    wb = read_workbook(path)

    # ---- 定位表头行：同时含「序号」「船号」「工序编码」「状态」 ----
    sheet, hi = None, -1
    for name, rows in wb.items():
        for i in range(min(20, len(rows))):
            if hi >= 0:
                break
            cells = [str(x).replace("\n", "").strip() for x in rows[i] if x is not None]
            if ("船号" in cells and "工序编码" in "".join(cells)
                    and "状态" in cells):
                sheet, hi = name, i
                break
        if hi >= 0:
            break
    if hi < 0:
        iss.add("", 0, "", "未找到排布计划详情表头（需含船号+工序编码+状态）",
                fname, "", "错误")
        iss.save(conn)
        return {"file": fname, "records": 0, "error": "表头不匹配"}

    headers = [str(x).replace("\n", "").strip() if x is not None else ""
               for x in wb[sheet][hi]]
    cm = _map_cols(headers, _PLAN_COLS)
    rows = wb[sheet]

    recs, seen = [], set()
    for ri in range(hi + 1, len(rows)):
        r = rows[ri]
        code = norm_text(cell(r, cm.get("工序编码")))
        if not code:
            continue
        ship_no = norm_ship_no(cell(r, cm.get("船号")))
        if not ship_no:
            continue
        key = (ship_no, code)
        if key in seen:
            continue
        seen.add(key)
        raw_status = norm_text(cell(r, cm.get("状态")))
        done = 1 if raw_status in PLAN_DONE_STATUS else 0
        quota_h = to_number(cell(r, cm.get("定额工时")), 0.0)
        remain_h = 0.0 if done else to_number(cell(r, cm.get("剩余工时")), 0.0)
        if not done and remain_h <= 0:
            remain_h = quota_h          # 剩余为空时退回定额，避免算出零工时
        recs.append((ship_no, code, norm_text(cell(r, cm.get("工序名称"))),
                     normalize_team(cell(r, cm.get("班组"))),
                     norm_text(cell(r, cm.get("工段"))),
                     quota_h, round(remain_h, 2),
                     norm_text(cell(r, find_col(headers, "计划完成日期")))[:30],
                     norm_text(cell(r, find_col(headers, "实际完成日期")))[:30],
                     raw_status, done, fname))

    if recs:
        conn.executemany("INSERT OR REPLACE INTO plan_detail"
                         "(ship_no,process_code,process_name,team,section,quota_hours,"
                         "remain_hours,plan_finish,actual_finish,raw_status,done,source_file)"
                         " VALUES(?,?,?,?,?,?,?,?,?,?,?,?)", recs)
        conn.commit()
    iss.save(conn)
    log(f"[{fname}] 排布计划详情：导入 {len(recs)} 道，"
        f"其中报完工 {sum(1 for x in recs if x[10])} 道")
    return {"file": fname, "records": len(recs),
            "done": sum(1 for x in recs if x[10])}


def merge_cross_progress(conn) -> dict:
    """
    交叉合并两个进度来源，判断每道工序到底做完了没有。

    规则（用户明确要求，三个来源**交替验证 / 取并集**）：
      * 班组周计划（初装1组…）显示 已完成/已完工/X  → 算完成
      * 单船工序完成情况表显示 已完工/加工中/…      → 算完成（加工中不再计剩余工时）
      * 排布计划详情显示 已完成/已完工/加工中/作业中 → 算完成
      * 任一方显示已完成，其它方显示未完成或空格 → **仍算完成**（取并集）
      * 三个来源都说"未开始" → 才算未完成
    """
    done_ap = {(r["ship_no"], r["process_code"]) for r in conn.execute(
        "SELECT ship_no,process_code FROM actual_progress WHERE status='已完成'")}
    done_rt = {(r["ship_no"], r["process_code"]) for r in conn.execute(
        "SELECT ship_no,process_code FROM realtime_progress WHERE done=1")}
    done_pl = {(r["ship_no"], r["process_code"]) for r in conn.execute(
        "SELECT ship_no,process_code FROM plan_detail WHERE done=1")}
    all_done = done_ap | done_rt | done_pl

    updated_done = updated_remain = 0
    rows = conn.execute("SELECT id,ship_no,process_code,planned_hours,status,"
                        "remaining_hours FROM tasks").fetchall()
    updates, remain_updates = [], []
    for r in rows:
        key = (r["ship_no"], r["process_code"])
        if key in all_done:
            # 注意：不能只判断 status!='已完成'。
            # 已经是"已完成"但 remaining_hours>0 的工序（状态与工时自相矛盾）
            # 否则永远修不好，排产校验会一直拒绝（A70L007 曾卡在这里）。
            if r["status"] != STATUS_DONE or float(r["remaining_hours"] or 0) > 0:
                updates.append((r["planned_hours"], r["id"]))
        else:
            # 三源都说未完成：若实时表/排布表给了剩余工时，用它覆盖（比"全额剩余"更准）
            rr = conn.execute(
                "SELECT remain_hours FROM realtime_progress WHERE ship_no=? AND process_code=?",
                (r["ship_no"], r["process_code"])).fetchone()
            if (rr is None or float(rr["remain_hours"] or 0) <= 0):
                rr = conn.execute(
                    "SELECT remain_hours FROM plan_detail WHERE ship_no=? AND process_code=?",
                    (r["ship_no"], r["process_code"])).fetchone()
            # 只用"大于 0"的剩余工时去修正 —— 否则读到 0 会把剩余量错误清零
            if rr is not None and float(rr["remain_hours"] or 0) > 0:
                remain_updates.append((float(rr["remain_hours"]), r["id"]))

    for planned, tid in updates:
        conn.execute("UPDATE tasks SET status=?,progress=1.0,done_hours=?,remaining_hours=0,"
                     "confidence=0.95,remarks='交叉核对：班组表/完成情况表/排布单已报完成' WHERE id=?",
                     (STATUS_DONE, planned, tid))
        updated_done += 1
    for rem, tid in remain_updates:
        conn.execute("UPDATE tasks SET remaining_hours=?,done_hours=MAX(0,planned_hours-?),"
                     "progress=CASE WHEN planned_hours>0 THEN MIN(1.0,MAX(0,planned_hours-?)/planned_hours) ELSE 0 END "
                     "WHERE id=?", (rem, rem, rem, tid))
        updated_remain += 1
    conn.commit()

    # 实时表里有、但 tasks 里没有的工序：按实时表的剩余工时补建
    created = 0
    miss = conn.execute(
        "SELECT rp.* FROM realtime_progress rp LEFT JOIN tasks t "
        "ON t.ship_no=rp.ship_no AND t.process_code=rp.process_code WHERE t.id IS NULL").fetchall()
    new_rows = []
    for rp in miss:
        if rp["done"] and (rp["remain_hours"] or 0) <= 0:
            continue                       # 已完成且无剩余 -> 无需建任务
        # 剩余为 0 时退回定额工时，避免补建出"零工时"的空任务
        rem = 0.0 if rp["done"] else float(
            (rp["remain_hours"] or 0) or (rp["quota_hours"] or 0))
        new_rows.append((rp["ship_no"], resolve_ship_type(rp["ship_no"]), rp["process_code"],
                         rp["process_name"], rp["team"], rp["section"], "", "通用",
                         float(rp["quota_hours"] or 0), 0.0, rem,
                         STATUS_DONE if rp["done"] else STATUS_TODO,
                         1.0 if rp["done"] else 0.0, 0.95, None, 1, 999, 1, 4, 0, "", 1,
                         "实时表", ""))
    if new_rows:
        conn.executemany(
            "INSERT OR IGNORE INTO tasks(ship_no,ship_type,process_code,process_name,team,"
            "section,zone,task_type,planned_hours,done_hours,remaining_hours,status,progress,"
            "confidence,actual_finish,earliest_day,latest_day,min_w,max_w,critical,milestone,"
            "material_ready,source,remarks) VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)",
            new_rows)
        conn.commit()
        created = len(new_rows)

    # 冲突统计：完成情况表说"未完成"、班组情况表说"已完成" -> 按并集规则判完成
    # 记录这个数，让用户能直接看到班组情况表确实在发挥作用
    conflict = conn.execute(
        "SELECT COUNT(*) FROM tasks t "
        "JOIN actual_progress ap ON ap.ship_no=t.ship_no AND ap.process_code=t.process_code "
        "JOIN realtime_progress rp ON rp.ship_no=t.ship_no AND rp.process_code=t.process_code "
        "WHERE ap.status=? AND rp.done=0", (STATUS_DONE,)).fetchone()[0]
    log(f"交叉核对：完成 {updated_done} 道、按实时表修正剩余 {updated_remain} 道、"
        f"补建 {created} 道（班组情况表完成 {len(done_ap)} + 完成情况表完成 {len(done_rt)}"
        f" + 排布单完成 {len(done_pl)}）；"
        f"其中 {conflict} 道是『完成情况表未完成、班组情况表已完成』，按班组表判完成")
    return {"done": updated_done, "remain_fixed": updated_remain, "created": created,
            "from_team_plan": len(done_ap), "from_realtime": len(done_rt),
            "from_plan_detail": len(done_pl),
            "team_fixed_conflict": conflict}


# ============================================================ 工序展开与进度合并
def fix_na_teams(conn):
    """班组是 N/A / 空的工序：用「生产周期」模板的班组补上。

    旧班组计划表建工序时组名解析失败会落成 N/A（如 A36MC008 有 67 道），
    这些工序还会挡住 expand_tasks —— 模板看到"已有同编码工序"就跳过，
    于是这艘船既用不上模板的班组、也拿不到生产日。
    修法：模板里能对上的，直接用模板的班组（顺带把生产日也对齐）。
    """
    try:
        n = conn.execute("""
            UPDATE tasks SET
              team = (SELECT tp.team FROM task_templates tp
                      WHERE tp.ship_type=tasks.ship_type
                        AND tp.process_code=tasks.process_code),
              earliest_day = COALESCE((SELECT tp.earliest_day FROM task_templates tp
                      WHERE tp.ship_type=tasks.ship_type
                        AND tp.process_code=tasks.process_code), earliest_day),
              latest_day = COALESCE((SELECT tp.latest_day FROM task_templates tp
                      WHERE tp.ship_type=tasks.ship_type
                        AND tp.process_code=tasks.process_code), latest_day)
            WHERE (team='N/A' OR team='' OR team IS NULL)
              AND EXISTS (SELECT 1 FROM task_templates tp
                          WHERE tp.ship_type=tasks.ship_type
                            AND tp.process_code=tasks.process_code
                            AND tp.team IS NOT NULL AND tp.team<>'')
        """).rowcount
        conn.commit()
    except Exception as e:
        log("补 N/A 班组失败：%s" % e)
        return {"fixed": 0}
    if n:
        log("班组为 N/A 的工序：按模板补齐 %d 道" % n)
    return {"fixed": n}


def apply_template_days(conn):
    """把「生产周期」模板里的生产日回填到实际工序（tasks）。

    【为什么必须有这一步】tasks 是靠导入进度表 / 报工记录逐条建的，
    建的时候拿不到模板的生产日，`earliest_day` 一律给了默认 1 ——
    实测全库 25891 道工序里 **21981 道（85%）都是 1**。
    排产按"第 N 个生产日"换算日期，于是所有工序都变成"第 1 天就该开工"，
    生产周期表里排得好好的先后顺序彻底失效（PMC 反馈"逻辑完全行不通"）。

    这里按 (ship_type, process_code) 把模板的 earliest_day / latest_day 刷回去。
    只影响能对上模板的工序；对不上的保持原值，不动它。
    """
    try:
        n = conn.execute("""
            UPDATE tasks SET
              earliest_day = (SELECT t.earliest_day FROM task_templates t
                              WHERE t.ship_type = tasks.ship_type
                                AND t.process_code = tasks.process_code),
              latest_day   = (SELECT t.latest_day FROM task_templates t
                              WHERE t.ship_type = tasks.ship_type
                                AND t.process_code = tasks.process_code)
            WHERE EXISTS (SELECT 1 FROM task_templates t
                          WHERE t.ship_type = tasks.ship_type
                            AND t.process_code = tasks.process_code)
        """).rowcount
        conn.commit()
    except Exception as e:
        log("回填模板生产日失败：%s" % e)
        return {"updated": 0, "error": str(e)}
    left = conn.execute("SELECT COUNT(*) c FROM tasks WHERE COALESCE(earliest_day,1)<=1"
                        ).fetchone()["c"]
    log("模板生产日回填：更新 %d 道工序；仍有 %d 道没有生产日（模板未覆盖）" % (n, left))
    return {"updated": n, "no_day": left}


def spread_unmapped_days(conn):
    """模板没覆盖到的工序：按"同船同班组的锚点区间"均匀铺开。

    「每日生产周期」只列主流程关键工序（A36S 286 道 / 整船 600+ 道），
    其余工序（泡沫预制件、选配件等）没有生产日。它们原本 earliest_day=1，
    会和"第 1 天"的关键工序一起抢产能 —— 实测 A36S266 第 1 天堆了 143 道，
    而第 8 天只有 1 道，关键工序反而被挤掉。

    这里按同一个 (船, 班组) 里"有生产日的工序"的区间 [最早, 最晚]，
    把这些无日期的工序按工序编码顺序均匀插进去 —— 既不会全挤第 1 天，
    也不会打乱关键工序的先后（关键工序的生产日一律不动）。
    """
    # 锚点 = 该 (船, 班组) 里【模板覆盖】的未完成工序的生产日
    # （注意和待均摊的查询分开：待均摊的是"模板没有"的工序，锚点必须从"模板有"的取）
    anchor_map = defaultdict(list)
    for r in conn.execute(
            "SELECT ship_no, team, earliest_day FROM tasks "
            "WHERE status<>'已完成' AND remaining_hours>0 AND earliest_day>1 "
            "AND EXISTS (SELECT 1 FROM task_templates tp "
            "            WHERE tp.ship_type=tasks.ship_type "
            "              AND tp.process_code=tasks.process_code)"):
        anchor_map[(r["ship_no"], r["team"])].append(int(r["earliest_day"] or 0))

    rows = conn.execute(
        "SELECT id, ship_no, team, process_code, earliest_day FROM tasks "
        "WHERE status<>'已完成' AND remaining_hours>0 "
        # ⚠ 只均摊"模板里确实没有"的工序。
        # 模板规定第 1 天的工序（船底调平、试装隔舱板等船体主线）earliest_day 也是 1，
        # 如果把它们也均摊掉，龙骨/隔舱板粘结等后续主线全被推后，整船顺序倒挂
        # （2026-10-29 实测 A32S102 初装倒挂 18 处就是这个原因）。
        "AND NOT EXISTS (SELECT 1 FROM task_templates tp "
        "                WHERE tp.ship_type=tasks.ship_type "
        "                  AND tp.process_code=tasks.process_code)"
    ).fetchall()
    groups = defaultdict(list)
    for r in rows:
        groups[(r["ship_no"], r["team"])].append(
            (str(r["process_code"] or ""), int(r["earliest_day"] or 0), r["id"]))
    ups, n_unm = [], 0
    for _key, items in groups.items():
        anchors = anchor_map.get(_key) or []
        if not anchors:
            continue                      # 这个班组本来就没生产日，保持原样
        lo, hi = min(anchors), max(anchors)
        if hi <= lo:
            continue
        unm = sorted([(c, i) for c, d, i in items if d <= 1])
        if not unm:
            continue
        n_unm += len(unm)
        n = len(unm)
        for k, (_c, tid) in enumerate(unm):
            d = lo + (hi - lo) * (k + 1) // (n + 1)
            ups.append((max(lo, min(hi, d)), tid))
    if ups:
        conn.executemany("UPDATE tasks SET earliest_day=? WHERE id=?", ups)
        conn.commit()
        log("无生产日工序：%d 道按班组区间均摊（都在关键工序之后）" % n_unm)
    return {"spread": len(ups)}


def expand_tasks(conn):
    ships = conn.execute("SELECT ship_no,ship_type FROM ships WHERE status='在制'").fetchall()
    added = 0
    no_template = set()
    for s in ships:
        st = s["ship_type"] or ""
        tmpl = conn.execute("SELECT * FROM task_templates WHERE ship_type=?", (st,)).fetchall()
        if not tmpl:
            no_template.add(st or "(未知)")
            continue
        exist = {r[0] for r in conn.execute(
            "SELECT process_code FROM tasks WHERE ship_no=?", (s["ship_no"],))}
        rows = []
        for t in tmpl:
            if t["process_code"] in exist:
                continue
            rows.append((s["ship_no"], st, t["process_code"], t["process_name"], t["team"],
                         t["section"], t["zone"], t["task_type"], t["standard_hours"], 0.0,
                         t["standard_hours"], STATUS_TODO, 0.0, 0.6, None, t["earliest_day"],
                         t["latest_day"], 1, 4, 0, t["milestone"] or "", 1, "模板", ""))
        if rows:
            conn.executemany(
                "INSERT INTO tasks(ship_no,ship_type,process_code,process_name,team,section,zone,"
                "task_type,planned_hours,done_hours,remaining_hours,status,progress,confidence,"
                "actual_finish,earliest_day,latest_day,min_w,max_w,critical,milestone,"
                "material_ready,source,remarks) VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)",
                rows)
            added += len(rows)
    conn.commit()
    log(f"工序展开：新增 {added} 道（{len(ships)} 艘在制船）")
    return {"tasks": added, "no_template": sorted(no_template)}


def sync_ships_from_progress(conn):
    # 班组情况表里出现过的船自动建档 —— 但界面上已"叉掉"（作废）的船不再复活
    blocked = _excluded_set(conn)
    ship_nos = [r[0] for r in conn.execute("SELECT DISTINCT ship_no FROM actual_progress")]
    added = fixed = 0
    for sno in ship_nos:
        if not sno:
            continue
        if str(sno).strip().upper() in blocked:
            continue
        row = conn.execute("SELECT ship_no,status,ship_type FROM ships WHERE ship_no=?", (sno,)).fetchone()
        if row is None:
            conn.execute("INSERT INTO ships(ship_no,ship_type,status,manual_priority,priority)"
                         " VALUES(?,?,'在制','正常',2)", (sno, resolve_ship_type(sno)))
            added += 1
        else:
            if row["status"] in ("完工", "未开工"):
                conn.execute("UPDATE ships SET status='在制' WHERE ship_no=?", (sno,))
                fixed += 1
            if not row["ship_type"]:
                conn.execute("UPDATE ships SET ship_type=? WHERE ship_no=?",
                             (resolve_ship_type(sno), sno))
    conn.commit()
    return {"added": added, "fixed": fixed}


def normalize_task_teams(conn):
    changed = 0
    rows = conn.execute("SELECT id,team FROM tasks").fetchall()
    for r in rows:
        if not r["team"]:
            continue
        nt = normalize_team(r["team"])
        if nt and nt != r["team"]:
            conn.execute("UPDATE tasks SET team=? WHERE id=?", (nt, r["id"]))
            changed += 1
    conn.commit()
    if changed:
        log(f"班组名称归一化 {changed} 道（机装/管装 -> 机管）")
    return changed


def sync_actual_progress(conn, doing_ratio=0.5):
    # 先体检：完成工时不可能大于定额
    conn.execute("UPDATE tasks SET done_hours=planned_hours "
                 "WHERE planned_hours>0 AND done_hours>planned_hours")
    conn.execute("UPDATE tasks SET remaining_hours=planned_hours "
                 "WHERE planned_hours>0 AND remaining_hours>planned_hours")
    conn.commit()

    rows = conn.execute("SELECT * FROM actual_progress").fetchall()
    agg = {}
    for r in rows:
        key = (r["ship_no"], r["process_code"])
        score = (1 if r["actual_finish"] else 0, 1 if r["raw_status"] else 0, r["id"])
        old = agg.get(key)
        if old is None or score > old[0]:
            agg[key] = (score, r)

    tasks = {(r["ship_no"], r["process_code"]): dict(r)
             for r in conn.execute("SELECT * FROM tasks")}
    updated = created = 0
    new_rows = []
    for (ship_no, code), (_s, ap) in agg.items():
        t = tasks.get((ship_no, code))
        hours = ap["hours"] or 0.0
        if t is None:
            rem = 0.0 if ap["status"] == STATUS_DONE else (
                hours * doing_ratio if ap["status"] == STATUS_DOING else hours)
            done = hours if ap["status"] == STATUS_DONE else (hours - rem)
            new_rows.append((ship_no, resolve_ship_type(ship_no), code, ap["process_name"],
                             ap["team"], ap["section"], "", "通用", hours, done, rem,
                             ap["status"], (done / hours) if hours else 0.0,
                             ap["confidence"], ap["actual_finish"], 1, 999, 1, 4, 0, "", 1,
                             "班组计划", (ap["not_done_reason"] or "")[:200]))
            created += 1
            continue
        tid = t["id"]
        if hours > 0 and abs(hours - (t["planned_hours"] or 0)) > 0.01:
            conn.execute("UPDATE tasks SET planned_hours=? WHERE id=?", (hours, tid))
        st = ap["status"]
        if st == STATUS_DONE:
            conn.execute("UPDATE tasks SET status=?,progress=1.0,done_hours=planned_hours,"
                         "remaining_hours=0 WHERE id=?", (STATUS_DONE, tid))
        elif st == STATUS_DOING:
            conn.execute("UPDATE tasks SET status=?,progress=?,done_hours=planned_hours*?,"
                         "remaining_hours=planned_hours*? WHERE id=?",
                         (STATUS_DOING, 1 - doing_ratio, (1 - doing_ratio), doing_ratio, tid))
        elif st in (STATUS_ABNORMAL, STATUS_FROZEN):
            conn.execute("UPDATE tasks SET status=?,remaining_hours=planned_hours WHERE id=?", (st, tid))
        elif t["status"] not in (STATUS_DONE, STATUS_DOING):
            conn.execute("UPDATE tasks SET status=?,progress=0,done_hours=0,"
                         "remaining_hours=planned_hours WHERE id=?", (STATUS_TODO, tid))
        conn.execute("UPDATE tasks SET actual_finish=?,confidence=? WHERE id=?",
                     (ap["actual_finish"], ap["confidence"] or 0.6, tid))
        if ap["not_done_reason"]:
            conn.execute("UPDATE tasks SET remarks=? WHERE id=?",
                         (ap["not_done_reason"][:200], tid))
        if not t["team"]:
            conn.execute("UPDATE tasks SET team=? WHERE id=?", (ap["team"], tid))
        updated += 1
    if new_rows:
        conn.executemany(
            "INSERT OR IGNORE INTO tasks(ship_no,ship_type,process_code,process_name,team,section,"
            "zone,task_type,planned_hours,done_hours,remaining_hours,status,progress,confidence,"
            "actual_finish,earliest_day,latest_day,min_w,max_w,critical,milestone,material_ready,"
            "source,remarks) VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)", new_rows)
    conn.commit()
    log(f"实际进度合并：更新 {updated} 道，新建 {created} 道")
    return {"updated": updated, "created": created}


def infer_implicit_completion(conn):
    deps = conn.execute("SELECT * FROM task_dependencies").fetchall()
    if not deps:
        return 0
    succ = defaultdict(list)
    for d in deps:
        succ[(d["ship_type"], d["predecessor_code"])].append(d["process_code"])
    tasks = {(r["ship_no"], r["process_code"]): dict(r) for r in conn.execute("SELECT * FROM tasks")}
    n = 0
    for t in tasks.values():
        if t["status"] == STATUS_DONE:
            continue
        for sc in succ.get((t["ship_type"], t["process_code"]), []):
            s = tasks.get((t["ship_no"], sc))
            if s and s["status"] == STATUS_DONE:
                conn.execute("UPDATE tasks SET status=?,progress=1.0,done_hours=planned_hours,"
                             "remaining_hours=0,confidence=0.7,remarks='后置已完成，推定本工序完成' "
                             "WHERE id=?", (STATUS_DONE, t["id"]))
                n += 1
                break
    conn.commit()
    if n:
        log(f"推定完成（后置已完工但未打卡）{n} 道")
    return n


def infer_by_production_day(conn):
    """同班组已施工到 D30，则 D30 之前的早期工序推定完成（解决周计划只保留近期作业的问题）"""
    rows = conn.execute(
        "SELECT t.* FROM tasks t JOIN ships s ON s.ship_no=t.ship_no "
        "WHERE s.status='在制'").fetchall()
    by = defaultdict(list)
    for r in rows:
        by[(r["ship_no"], r["team"])].append(dict(r))
    n = 0
    for (ship_no, team), ts in by.items():
        done_days = [t["latest_day"] for t in ts
                     if t["status"] == STATUS_DONE and t["latest_day"] and t["latest_day"] <= 400]
        if not done_days:
            continue
        dmax = max(done_days)
        for t in ts:
            if t["status"] == STATUS_DONE:
                continue
            if t["latest_day"] and t["latest_day"] <= 400 and t["latest_day"] < dmax:
                conn.execute("UPDATE tasks SET status=?,progress=1.0,done_hours=planned_hours,"
                             "remaining_hours=0,confidence=0.5,remarks=? WHERE id=?",
                             (STATUS_DONE, f"按生产日推定完成（{team}已施工至 D{dmax}）", t["id"]))
                n += 1
    conn.commit()
    if n:
        log(f"按标准生产日推定完成 {n} 道")
    return n


# 班组情况表里表示"已经做完"的状态值（含用户提到的 X 项）
TEAM_DONE_STATUS = ("已完成", "已完工", "完工", "完成", "X", "x")


def sync_missing_from_team_plan(conn):
    """
    班组情况表里有、但系统里还没有这道工序的 —— 补建任务。

    否则像 A70E5196 / A70E5198 这种只在班组表出现的活会被漏掉（少算工时）。
    只补【状态空白（未做）且工时>0】的，已完成的不补。
    """
    stype = {r["ship_no"]: r["ship_type"] for r in conn.execute(
        "SELECT ship_no, ship_type FROM ships")}
    seen = set()
    rows = []
    for r in conn.execute(
            "SELECT ap.ship_no, ap.process_code, ap.process_name, ap.team, ap.hours, "
            "ap.status, ap.raw_status, ap.section FROM actual_progress ap "
            "WHERE (ap.raw_status IS NULL OR ap.raw_status='') AND ap.hours>0 "
            "AND NOT EXISTS(SELECT 1 FROM tasks t WHERE t.ship_no=ap.ship_no "
            "  AND t.process_code=ap.process_code) "
            "ORDER BY ap.ship_no, ap.process_code"):
        key = (r["ship_no"], r["process_code"])
        if key in seen or r["ship_no"] not in stype:
            continue
        seen.add(key)
        h = float(r["hours"] or 0)
        rows.append((r["ship_no"], stype[r["ship_no"]], r["process_code"],
                     r["process_name"] or "", r["team"] or "", r["section"] or "",
                     h, h, STATUS_TODO, 0.0, 0.9, 1, "班组情况表"))
    if not rows:
        return 0
    conn.executemany(
        "INSERT INTO tasks(ship_no,ship_type,process_code,process_name,team,section,"
        "planned_hours,remaining_hours,status,progress,confidence,earliest_day,source) "
        "VALUES(?,?,?,?,?,?,?,?,?,?,?,?,'班组情况表')"
        if False else
        "INSERT INTO tasks(ship_no,ship_type,process_code,process_name,team,section,"
        "planned_hours,remaining_hours,status,progress,confidence,earliest_day,source) "
        "VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?)", rows)
    conn.commit()
    log(f"班组情况表独有工序补建：{len(rows)} 道")
    return len(rows)


def apply_final_remaining(conn):
    """
    【最终剩余工时】规则（PMC 口径，以班组情况表为准）：

    1. 选配项：只保留 S / su / 空值；O 和 0 的剔除
    2. 班组情况表状态为 已完成 / 已完工 / X 的剔除
    3. 工序工位完成情况表已完工、但班组情况表没标已完成的 —— 也剔除
       （即：完成情况表说完工的，一律不算剩余工时）
    4. 剩余工时 = min(班组情况表工时, 工序工位完成情况表剩余工时) —— **取小的那个**
    5. 班组整体未启动（班组表一条都没有）的，走 fill_unstarted_teams 按满额，这里不动
    """
    started = _started_ship_teams(conn)
    # 一次性载入内存（逐工序写子查询会让两万道工序跑几十秒）
    ap_map = {}
    # 同一道工序在班组表里可能有多条记录（不同周次），去重时必须【完成状态优先】：
    # 只要有一条标了 已完工/已完成/X，整道工序就算做完；否则会漏掉 X 而多算工时。
    # 同时保留 raw_status：Excel 的 "X" 会被规范化成"未开始"，只看 status 会误判。
    def _done(status, raw):
        return (any(k in str(status or "") for k in TEAM_DONE_STATUS) or
                any(k in str(raw or "") for k in TEAM_DONE_STATUS))
    for r in conn.execute(
            "SELECT ship_no, process_code, status, hours, option_flag, raw_status "
            "FROM actual_progress"):
        cur = (r["status"], r["hours"], r["option_flag"], r["raw_status"])
        key = (r["ship_no"], r["process_code"])
        old = ap_map.get(key)
        if old is None:
            ap_map[key] = cur
        elif _done(old[0], old[3]) and not _done(cur[0], cur[3]):
            # 【空白优先】PMC 的做法是在班组表里"筛掉已完成/已完工/X，只看空白"：
            # 同一道工序只要还有一行是空白，就按没做完算（空白行的工时为准）
            ap_map[key] = cur
    rt_map = {}
    for r in conn.execute(
            "SELECT ship_no, process_code, done, remain_hours FROM realtime_progress"):
        rt_map[(r["ship_no"], r["process_code"])] = (r["done"], r["remain_hours"])

    # 注意：这里【不排除 status='已完成'】—— 班组表还是空白的工序，
    # 即使之前被完成情况表判成完工，也要按班组表口径恢复成"没做完"
    rows = conn.execute(
        "SELECT id, ship_no, team, process_code, planned_hours, status FROM tasks"
    ).fetchall()

    # 另外两个交替验证源报完工的工序集合（班组表没覆盖的船靠它们判定）
    rt_done = {(r["ship_no"], r["process_code"]) for r in conn.execute(
        "SELECT ship_no,process_code FROM realtime_progress WHERE done=1")}
    pl_done = {(r["ship_no"], r["process_code"]) for r in conn.execute(
        "SELECT ship_no,process_code FROM plan_detail WHERE done=1")}

    mode = str(get_setting(conn, "remaining_source", "min") or "min").lower()
    zero, keep, hours, reopened = 0, 0, 0.0, 0
    sum_min = sum_team = sum_rt = 0.0
    updates = []
    for r in rows:
        key = (r["ship_no"], r["process_code"])
        if (r["ship_no"], r["team"]) not in started:
            # 班组情况表没覆盖这个(船,班组)。按用户"三源交替覆盖"口径：
            # 单船完成情况表 / 排布计划详情 只要有一方报完工，就算完成、不计工时。
            if key in rt_done or key in pl_done:
                updates.append((0.0, r["id"])); zero += 1
                if str(r["status"]) != STATUS_DONE:
                    conn.execute(
                        "UPDATE tasks SET status=?, progress=1.0, done_hours=?, "
                        "confidence=0.95, remarks='交替验证：完成情况表或排布单已报完成' "
                        "WHERE id=?", (STATUS_DONE, r["planned_hours"], r["id"]))
                continue
            # 三个来源都没说完工 -> 保持 fill_unstarted_teams 的满额。
            # 但状态若残留"已完成"必须改回未开始：否则会出现
            # 「已完成 + 有剩余工时」自相矛盾，排产校验直接拒绝、卡死无法排产
            # （A70L007 五个班组都不在班组表里，197 道卡在这里）。
            if str(r["status"]) == STATUS_DONE:
                conn.execute("UPDATE tasks SET status=?, progress=0, done_hours=0 "
                             "WHERE id=?", (STATUS_TODO, r["id"]))
                reopened += 1
            continue
        # 【三源交替验证·取并集】排布计划详情 报了 已完成/已完工/加工中/作业中/…
        # 就算这道工序完工、不计入工时。注意：班组表空白不能覆盖它 —— 空白只是"没说"，
        # 不是"说没做完"。否则排布单里已报完工的工序会被班组表空白顶回"未开始"、
        # 工时照算，排布单这个源等于白导入。
        # （单船工序完成情况表仍保持既有口径：班组表空白时按班组表算，见下方注释）
        if key in pl_done:
            updates.append((0.0, r["id"])); zero += 1
            if str(r["status"]) != STATUS_DONE:
                conn.execute(
                    "UPDATE tasks SET status=?, progress=1.0, done_hours=?, "
                    "confidence=0.95, remarks='交替验证：排布计划详情已报完成' "
                    "WHERE id=?", (STATUS_DONE, r["planned_hours"], r["id"]))
            continue
        # 班组已启动、但班组情况表里【没有这道工序】—— 按规则不计入
        # （用户的筛选方法就是在班组表里筛，表里没有的自然不算）
        if key not in ap_map:
            updates.append((0.0, r["id"])); zero += 1; continue
        ap_st, ap_h, ap_opt, raw_st = ap_map[key]
        rp_done, rp_rem = rt_map.get(key, (0, None))

        opt = str(ap_opt or "").strip()
        if opt.lower() in OPTION_SKIP_VALUES:
            updates.append((0.0, r["id"])); zero += 1; continue

        ap_st = str(ap_st or "").strip()
        # 是否做完【只看班组情况表的状态/原始值】(PMC 就是在班组表里筛 已完成/已完工/X)
        # 注意：不能再用"完成情况表说完工"去剔除 —— 班组表空白的活，
        # 即使完成情况表标了完工，PMC 的筛选结果里仍然算未做（实测 A70 初装差 14H 就是这原因）
        done_by_team = (any(k in ap_st for k in TEAM_DONE_STATUS) or
                        any(k in str(raw_st or "") for k in TEAM_DONE_STATUS))
        if done_by_team:
            updates.append((0.0, r["id"])); zero += 1; continue

        # 班组表空白 = 没做完：若之前被完成情况表判成"已完成"，这里恢复成未开始
        if str(r["status"]) == "已完成":
            conn.execute("UPDATE tasks SET status=?, progress=0, done_hours=0 "
                         "WHERE id=?", (STATUS_TODO, r["id"]))
            reopened += 1

        # 三种口径都算出来，便于和手工统计对账（实际写入哪个由 remaining_source 决定）
        ah = rr_ = 0.0
        try:
            ah = float(ap_h) if ap_h is not None else 0.0
        except (TypeError, ValueError):
            ah = 0.0
        try:
            rr_ = float(rp_rem) if rp_rem is not None else 0.0
        except (TypeError, ValueError):
            rr_ = 0.0
        both = [x for x in (ah, rr_) if x > 0]
        v_min = round(min(both), 2) if both else float(r["planned_hours"] or 0)
        v_team = ah if ah > 0 else (rr_ or float(r["planned_hours"] or 0))
        v_rt = rr_ if rr_ > 0 else (ah or float(r["planned_hours"] or 0))
        sum_min += v_min
        sum_team += v_team
        sum_rt += v_rt

        val = {"min": v_min, "team": v_team, "realtime": v_rt}.get(mode, v_min)
        updates.append((round(val, 2), r["id"]))
        keep += 1
        hours += val

    if updates:
        conn.executemany("UPDATE tasks SET remaining_hours=? WHERE id=?", updates)
        conn.commit()
    log(f"最终剩余工时（口径={mode}）：计入 {keep} 道共 {round(hours,1)} H，"
        f"剔除/归零 {zero} 道")
    log(f"  三种口径合计：取小值 {round(sum_min,1)} H ｜ 只看班组表 {round(sum_team,1)} H "
        f"｜ 只看完成情况表 {round(sum_rt,1)} H")
    return {"kept": keep, "hours": round(hours, 1), "dropped": zero, "mode": mode,
            "totals": {"min": round(sum_min, 1), "team": round(sum_team, 1),
                       "realtime": round(sum_rt, 1)}}


def _started_ship_teams(conn):
    """班组情况表里出现过的 (船号, 班组) 组合 —— 这些班组视为已启动"""
    return {(r["ship_no"], r["team"]) for r in conn.execute(
        "SELECT DISTINCT ship_no, team FROM actual_progress")}


def fill_unstarted_teams(conn):
    """
    班组整体还没启动 -> 工时按【100% 满额】计算。

    判定：该船该班组在班组情况表里一条记录都没有（不是"有记录但某工序没列"）。
    满额工时优先取该船自己的定额总工时；若自己没有（数据缺失），
    就取同船型其他船在该班组的满额工时（例如 A42C017 参考 A42C018）。
    """
    started = _started_ship_teams(conn)
    # 每船每班组的满额（定额总和）与工序数
    quota, cnt = {}, {}
    for r in conn.execute(
            "SELECT ship_no, ship_type, team, SUM(planned_hours) q, COUNT(*) n FROM tasks "
            "GROUP BY ship_no, ship_type, team"):
        quota[(r["ship_no"], r["team"])] = float(r["q"] or 0)
        cnt[(r["ship_no"], r["team"])] = int(r["n"] or 0)
    # 同船型每班组的参考满额：取【中位数】
    # 用最大值会被个别配置特殊的船拉高（例如 A42C 机管最大 429H、多数只有 271~415H），
    # 中位数更能代表"同型船的标准满额"。
    _vals = {}
    # 只拿【已启动】组合的满额做参考：
    # 否则未启动船被填充后的值会反过来混进参考值里，越算越大（A42C 终装曾被推到 1392H）
    stype = {r["ship_no"]: r["ship_type"] for r in conn.execute(
        "SELECT ship_no, ship_type FROM ships")}
    for (s, t) in started:
        q = quota.get((s, t), 0.0)
        if q > 0 and s in stype:
            _vals.setdefault((stype[s], t), []).append(q)
    ref = {}
    for k, v in _vals.items():
        v.sort()
        ref[k] = v[len(v) // 2]

    done = []
    for r in conn.execute(
            "SELECT ship_no, ship_type, team FROM tasks "
            "WHERE status<>'已完成' GROUP BY ship_no, ship_type, team"):
        key = (r["ship_no"], r["team"])
        if key in started:
            continue                       # 已启动的班组不走这里
        q = quota.get(key, 0.0)
        n = cnt.get(key, 0)
        ref_q = ref.get((r["ship_type"], r["team"]), 0.0)
        # 优先用【本船自己的满额定额】：它反映这条船的真实配置
        # （实测 A42C 各船终装：111 道=822H、215 道=1392H、285 道=1491H、334 道≈1700H，
        #   工序越全满额越大，所以不能用同型中位数去套）。
        # 只有本船连定额都没有时，才退回同船型参考满额（用户说的"拿 A42C018 的满额算"）。
        if q > 0:
            conn.execute(
                "UPDATE tasks SET remaining_hours=planned_hours "
                "WHERE ship_no=? AND team=? AND status<>'已完成'",
                (r["ship_no"], r["team"]))
            done.append((r["ship_no"], r["team"], round(q, 1), "本船满额定额"))
        elif ref_q > 0 and n > 0:
            each = round(ref_q / n, 2)
            conn.execute(
                "UPDATE tasks SET planned_hours=?, remaining_hours=? "
                "WHERE ship_no=? AND team=? AND status<>'已完成'",
                (each, each, r["ship_no"], r["team"]))
            done.append((r["ship_no"], r["team"], round(each * n, 1),
                         f"同船型{r['ship_type']}参考满额"))
    if done:
        conn.commit()
        total = round(sum(x[2] for x in done), 1)
        log(f"未启动班组按 100% 满额计算：{len(done)} 个（船-班组）组合，共 {total} H")
    return done


def apply_team_scope(conn, enabled=None):
    """
    【班组已启动】但班组情况表里没列这道工序 -> 该工序不统计（剩余归零）。

    注意两点：
      1. 只归零 remaining_hours，**保留 planned_hours** 作为满额记录，
         否则"未启动班组"就没有满额数据可用了。
      2. 整个班组一条记录都没有的船-班组组合不算这里，由 fill_unstarted_teams 按满额处理。
    """
    if enabled is None:
        enabled = int(float(get_setting(conn, "team_scope_only", 1) or 1))
    if not enabled:
        return 0, 0.0
    started = _started_ship_teams(conn)
    rows = conn.execute(
        "SELECT t.id, t.ship_no, t.team, t.remaining_hours FROM tasks t "
        "WHERE t.status<>'已完成' AND t.remaining_hours>0 AND NOT EXISTS("
        " SELECT 1 FROM actual_progress ap "
        " WHERE ap.ship_no=t.ship_no AND ap.process_code=t.process_code)").fetchall()
    hits = [r for r in rows if (r["ship_no"], r["team"]) in started]
    if not hits:
        return 0, 0.0
    saved = round(sum(float(r["remaining_hours"] or 0) for r in hits), 1)
    conn.executemany("UPDATE tasks SET remaining_hours=0 WHERE id=?",
                     [(r["id"],) for r in hits])
    conn.commit()
    log(f"班组情况表未列的工序不统计：{len(hits)} 道剩余归零，释放 {saved} H")
    return len(hits), saved


def apply_option_flag(conn):
    """
    选配项 = O / 0 的工序【不做时间统计】：
    工时与剩余工时归零（但保留工序记录，报表里看得到），于是不会占用产能、不参与排产。

    规则来源：班组情况表（如 初装1组-总装1线.xlsx）里的「选配项」列。
    """
    vals = list(OPTION_SKIP_VALUES)
    ph = ",".join("?" * len(vals))
    rows = conn.execute(
        "SELECT t.id, t.planned_hours FROM tasks t "
        "JOIN actual_progress ap ON ap.ship_no=t.ship_no AND ap.process_code=t.process_code "
        "WHERE t.status<>'已完成' AND LOWER(TRIM(COALESCE(ap.option_flag,''))) IN (%s)" % ph,
        vals).fetchall()
    if not rows:
        return 0, 0.0
    saved = round(sum(float(r["planned_hours"] or 0) for r in rows), 1)
    conn.executemany(
        "UPDATE tasks SET planned_hours=0, remaining_hours=0 WHERE id=?",
        [(r["id"],) for r in rows])
    conn.commit()
    log(f"选配项(O/0)不统计：{len(rows)} 道工序工时归零，释放 {saved} H")
    return len(rows), saved


def skip_expired_ships(conn, today=None, enabled=None):
    """
    逾期船【默认不再自动退出排产】。

    业务口径（PMC 评审确认）：**交检日已过 ≠ 已交检**。
    只有实际交检完成、订单取消或经批准关闭，才退出排产；
    逾期未完成的应继续留在计划里，并单独列为"已逾期在制"。

    因此默认值从 1 改成 0；若确实要按历史数据过滤，
    可在【系统设置】里把"交检日已过的船不参与排产"打开。
    """
    if enabled is None:
        enabled = int(float(get_setting(conn, "skip_expired_ships", 0) or 0))
    today = today or dt.date.today()
    if not enabled:
        return 0
    cur = conn.execute(
        "UPDATE ships SET status='已过期' "
        "WHERE status='在制' AND plan_inspection IS NOT NULL AND plan_inspection<>'' "
        "AND DATE(SUBSTR(plan_inspection,1,10)) < DATE(?)", (today.isoformat(),))
    conn.commit()
    n = cur.rowcount or 0
    if n:
        log(f"已跳过计划交检日期早于 {today} 的船 {n} 艘（状态=已过期，"
            f"可在【系统设置 → 船只】里改回来）")
    return n


def restore_expired_ships(conn, ship_nos=None):
    """把【已过期】的船恢复为在制（可指定船号，不传则全部恢复）"""
    if ship_nos:
        n = 0
        for s in ship_nos:
            n += conn.execute(
                "UPDATE ships SET status='在制' WHERE ship_no=? AND status='已过期'",
                (s,)).rowcount or 0
    else:
        cur = conn.execute("UPDATE ships SET status='在制' WHERE status='已过期'")
        n = cur.rowcount or 0
    conn.commit()
    if n:
        log(f"已恢复 {n} 艘被跳过的船为在制")
    return n


def expired_ships(conn):
    """列出被跳过的船"""
    return [dict(r) for r in conn.execute(
        "SELECT ship_no,ship_type,plan_inspection,current_stage,progress "
        "FROM ships WHERE status='已过期' ORDER BY plan_inspection")]


def mark_ship_status(conn):
    today = dt.date.today()
    ships = conn.execute("SELECT * FROM ships").fetchall()
    remain = {r[0]: (r[1] or 0) for r in conn.execute(
        "SELECT ship_no,SUM(remaining_hours) FROM tasks GROUP BY ship_no")}
    done = active = no_data = stale = 0
    for s in ships:
        sno, status = s["ship_no"], s["status"]
        if status in ("暂缓", "冻结"):
            continue
        insp = to_date(s["plan_inspection"])
        if insp and insp < today - dt.timedelta(days=60) and (s["progress"] or 0) <= 0.001:
            if status != "完工":
                conn.execute("UPDATE ships SET status='完工' WHERE ship_no=?", (sno,))
                stale += 1
            continue
        if sno not in remain:
            if status == "在制":
                conn.execute("UPDATE ships SET status='未开工' WHERE ship_no=?", (sno,))
                no_data += 1
            continue
        r = remain[sno]
        if r <= 0.01:
            if status != "完工":
                conn.execute("UPDATE ships SET status='完工' WHERE ship_no=?", (sno,))
                done += 1
        elif status != "在制":
            conn.execute("UPDATE ships SET status='在制' WHERE ship_no=?", (sno,))
            active += 1
    conn.commit()
    return {"done": done, "active": active, "no_data": no_data, "stale": stale}


def compute_ship_progress(conn):
    ships = conn.execute("SELECT ship_no FROM ships WHERE status='在制'").fetchall()
    for s in ships:
        row = conn.execute(
            "SELECT SUM(planned_hours) p, SUM(done_hours) d, "
            "SUM(CASE WHEN status<>'已完成' THEN remaining_hours ELSE 0 END) r "
            "FROM tasks WHERE ship_no=?", (s["ship_no"],)).fetchone()
        p, d, r = row["p"] or 0, row["d"] or 0, row["r"] or 0
        prog = min(1.0, (d / p)) if p > 0 else 0.0
        stage = conn.execute(
            "SELECT team,SUM(remaining_hours) h FROM tasks WHERE ship_no=? AND status='未开始' "
            "GROUP BY team ORDER BY h DESC LIMIT 1", (s["ship_no"],)).fetchone()
        conn.execute("UPDATE ships SET done_hours=?,remaining_hours=?,progress=?,current_stage=? "
                     "WHERE ship_no=?", (round(d, 1), round(r, 1), round(prog, 3),
                                         stage["team"] if stage else "未开工", s["ship_no"]))
    conn.commit()
    return len(ships)


# ============================================================ 一键导入
def run_full_import(conn):
    # 导入前先把桌面上更新的原始文件同步进 data/raw（按修改时间覆盖同名文件）。
    # 这样用户只需点「开始导入」一步：桌面换了新的班组周计划也能吃到最新数据，
    # 不必先点「从桌面复制」，避免"复制过了但导入的还是旧表"这类困惑。
    synced = []
    try:
        synced = copy_from_desktop()
        if synced:
            log(f"导入前已从桌面同步 {len(synced)} 个更新文件：{'、'.join(synced)}")
    except Exception as e:
        log(f"桌面同步失败，改用 data/raw 现有文件：{e}")
    files = scan_raw_files()
    conn.execute("DELETE FROM data_quality")
    conn.commit()
    stats = {"templates": [], "teams": [], "master": {}, "synced": synced,
             "plan_detail": []}

    # ---- 排布计划详情（第三个交替验证源，与班组表 / 单船完成情况表取并集）----
    for p in files.get("plan_detail", []):
        try:
            stats["plan_detail"].append(import_plan_detail(conn, p))
        except Exception as e:
            log(f"排布计划详情导入失败 {os.path.basename(p)}: {e}")
            stats["plan_detail"].append({"file": os.path.basename(p), "error": str(e)})

    for p in files["templates"]:
        try:
            stats["templates"].append(import_cycle_template(conn, p))
        except Exception as e:
            log(f"模板导入失败 {os.path.basename(p)}: {e}")
            stats["templates"].append({"ship_type": os.path.basename(p), "error": str(e)})

    if files["master_plan"]:
        try:
            stats["master"] = import_master_plan(conn, files["master_plan"])
        except Exception as e:
            log(f"主计划导入失败: {e}")
            stats["master"] = {"error": str(e)}

    for p in files["team_plans"]:
        try:
            stats["teams"].append(import_team_plan(conn, p))
        except Exception as e:
            log(f"班组导入失败 {os.path.basename(p)}: {e}")
            stats["teams"].append({"team": os.path.basename(p), "error": str(e)})

    # ---- 单船实时进度表（9-24 这类）----
    # 清掉历史上被误灌进来的模板数据（船型《生产周期》不是完成情况表）
    bad = conn.execute(
        "DELETE FROM realtime_progress WHERE source_file LIKE '%生产周期%' "
        "OR source_file LIKE '%清单%'").rowcount or 0
    if bad:
        conn.commit()
        log(f"已清除 {bad} 条误导入的模板记录（来自《生产周期》类文件）")

    rt_files = scan_realtime_files(conn)
    stats["realtime"] = []
    for p in rt_files:
        try:
            # 先按内容判断是不是"单船完成情况表"，避免模板混进来
            kind = classify_excel(p)
            if kind != "realtime":
                log(f"跳过非完成情况表：{os.path.basename(p)}（识别为 {kind}）")
                stats["realtime"].append({"file": os.path.basename(p), "skipped": kind})
                continue
            stats["realtime"].append(import_realtime_progress(conn, p))
        except Exception as e:
            log(f"实时表导入失败 {os.path.basename(p)}: {e}")
            stats["realtime"].append({"file": os.path.basename(p), "error": str(e)})

    # 合并大小写不一致造成的重复船号（A28mc068 / A28MC068）
    stats["ships_merged"] = merge_duplicate_ships(conn)
    stats["ships_from_progress"] = sync_ships_from_progress(conn)
    stats["tasks"] = expand_tasks(conn)
    stats["progress"] = sync_actual_progress(conn)
    stats["team_normalized"] = normalize_task_teams(conn)
    # 把「生产周期」模板的生产日回填到实际工序（否则 85% 的工序都是"第1天"，
    # 排产就不会按生产周期表的先后顺序走）
    stats["fix_na_teams"] = fix_na_teams(conn)          # 班组 N/A 的工序先按模板补齐
    stats["template_days"] = apply_template_days(conn)
    stats["spread_days"] = spread_unmapped_days(conn)
    # 交叉核对：周计划 + 实时表 取并集判断完成
    stats["cross"] = merge_cross_progress(conn)
    stats["inferred"] = infer_implicit_completion(conn)
    stats["inferred_by_day"] = infer_by_production_day(conn)
    # 先按"未启动班组 = 100% 满额"补齐，再剔除"已启动班组里没列的工序"
    unstarted = fill_unstarted_teams(conn)
    stats["unstarted_teams"] = [{"ship_no": s, "team": t, "hours": h, "source": src}
                                for (s, t, h, src) in unstarted]
    n_opt, h_opt = apply_option_flag(conn)
    stats["option_skipped"] = {"tasks": n_opt, "hours": h_opt}
    n_sc, h_sc = apply_team_scope(conn)
    stats["team_scope_skipped"] = {"tasks": n_sc, "hours": h_sc}
    # 最后按 PMC 口径重算：先补班组表独有的工序，再剔除选配O/0 与已完成类，剩余取两表较小值
    stats["team_only_created"] = sync_missing_from_team_plan(conn)
    stats["final_remaining"] = apply_final_remaining(conn)
    stats["ship_status"] = mark_ship_status(conn)
    # 手填的交检日优先于主计划（否则下次导入会把用户改的值冲掉）
    n_manual = conn.execute(
        "UPDATE ships SET plan_inspection=manual_inspection "
        "WHERE manual_inspection IS NOT NULL AND manual_inspection<>''").rowcount or 0
    if n_manual:
        conn.commit()
        log(f"已应用 {n_manual} 条手填交检日（优先于主计划）")
    stats["manual_inspection"] = n_manual
    # 口径调整迁移：旧版把逾期船标成"已过期"并排除在排产外。
    # 新口径是"交检日已过 ≠ 已交检"，所以这里把它们恢复成在制（仍会带【已逾期】标记）。
    if not int(float(get_setting(conn, "skip_expired_ships", 0) or 0)):
        _n = conn.execute(
            "UPDATE ships SET status='在制' WHERE status='已过期'").rowcount or 0
        if _n:
            conn.commit()
            log(f"已恢复 {_n} 艘此前被标记为过期的船（口径：交检日已过 ≠ 已交检）")
        stats["expired_restored"] = _n
    stats["expired_skipped"] = skip_expired_ships(conn)
    compute_ship_progress(conn)
    stats["in_production_ships"] = conn.execute(
        "SELECT COUNT(*) c FROM ships WHERE status='在制'").fetchone()["c"]
    log(f"导入完成：在制船 {stats['in_production_ships']} 艘")
    return stats

# ==========================================================================
#  模块：aps_schedule.py
# ==========================================================================
import datetime as dt
import json
import math
import os
from collections import defaultdict, Counter



# ============================================================ 工作日历
def ensure_calendar(conn, start: dt.date, days: int, extra_holidays=None, cfg=None):
    """
    生成/刷新工作日历（含加班安排）。

    规则：
      * 周六 / 周日是否上班由配置决定（PMC 可在界面改）
      * 每周加班 N 天、每次 M 小时：把加班分配到该周的前 N 个工作日
      * 界面里手工添加的节假日永远放假（不会被自动覆盖）
      * 每次重算只清除"自动生成"的记录，手工记录保留
    """
    cfg = dict(cfg or get_worktime_config(conn))

    end = start + dt.timedelta(days=days - 1)
    manual = {r['cal_date']:dict(r) for r in conn.execute("SELECT * FROM calendar WHERE remarks='手工'")}
    holidays = set(DEFAULT_HOLIDAYS) | {str(x)[:10] for x in (extra_holidays or [])}
    # 刷新范围内的【全部】记录，避免手工残留与新生成记录撞唯一键
    conn.execute("DELETE FROM calendar WHERE cal_date BETWEEN ? AND ?",
                 (start.isoformat(), end.isoformat()))

    rows = []
    week_ot = {}                       # (年,周) -> 本周已安排的加班天数
    d = start
    while d <= end:
        iso = d.isoformat()
        if iso in manual:
            m=manual[iso]
            rows.append((iso,m['day_type'],m['is_prod'],m['normal_hours'],m['overtime'],'手工'))
        elif iso in holidays:
            rows.append((iso, "节假日", 0, cfg["sh_per_day"], 0.0, "自动生成"))
        else:
            wd = d.weekday()
            is_work = (wd < 5) or (wd == 5 and cfg["work_sat"]) or (wd == 6 and cfg["work_sun"])
            if not is_work:
                rows.append((iso, "周末", 0, cfg["sh_per_day"], 0.0, "自动生成"))
            else:
                wk = (d.isocalendar()[0], d.isocalendar()[1])
                used = _weekday_work_index(conn, d, cfg) - 1
                ot = float(cfg["ot_hours"]) if used < int(cfg["ot_days_per_week"]) else 0.0
                if ot:
                    week_ot[wk] = used + 1
                rows.append((iso, "工作日", 1, cfg["sh_per_day"] + ot, ot, "自动生成"))
        d += dt.timedelta(days=1)

    if rows:
        conn.executemany("INSERT INTO calendar(cal_date,day_type,is_prod,normal_hours,"
                         "overtime,remarks) VALUES(?,?,?,?,?,?)", rows)
        conn.commit()
    return len(rows)


def load_teams(conn):
    return [dict(r) for r in conn.execute("SELECT * FROM teams ORDER BY sort_order")]


def team_capacity(conn, team: dict, d: dt.date, cal=None, tc=None, scenario=None, cfg=None):
    """
    计算某班组某天的产能 -> (人数, 有效工时, 是否上班)

    产能 = 熟练工 × (熟练工日工时 + 当天加班) + 新人 × (新人日工时 + 新人是否加班 ? 加班 : 0)
    例：终装 12×11 + 4×4 = 148H；若当天加班 3H 且新人加班 → 12×14 + 4×7
    """
    if cfg is None:
        cfg = get_worktime_config(conn)
    if cal is None:
        r = conn.execute("SELECT * FROM calendar WHERE cal_date=?", (d.isoformat(),)).fetchone()
        cal = dict(r) if r else None
    if cal and not cal["is_prod"]:
        return 0.0, 0.0, False

    work_sat = bool((scenario or {}).get("work_sat", team.get("work_sat", cfg["work_sat"])))
    work_sun = bool((scenario or {}).get("work_sun", team.get("work_sun", cfg["work_sun"]))) or bool((scenario or {}).get("work_sunday"))
    if d.weekday() == 5 and not work_sat:
        return 0.0, 0.0, False
    if d.weekday() == 6 and not work_sun:
        return 0.0, 0.0, False

    if tc is None:
        r = conn.execute("SELECT * FROM team_calendar WHERE cal_date=? AND team=?",
                         (d.isoformat(), team["team_code"])).fetchone()
        tc = dict(r) if r else None
    if tc is not None and not tc["is_working"]:
        return 0.0, 0.0, False

    skilled, new = int(team["skilled"]), int(team["new_workers"])
    if scenario:
        delta = scenario.get("team_delta", {}).get(team["team_code"], {})
        fd = to_date(delta.get("from_date"))
        if fd is None or d >= fd:
            skilled += int(delta.get("skilled", 0))
            new += int(delta.get("new", 0))

    # 当天加班小时：优先取日历（已按"每周加班 N 天"分配好）
    ot = float(cal.get("overtime") or 0) if cal else 0.0
    if tc is not None and tc.get("overtime"):
        ot = max(ot, float(tc["overtime"] or 0))
    if team.get("overtime"):
        ot = max(ot, float(team["overtime"] or 0))
    if scenario and scenario.get("ot_hours") is not None and (cal and cal.get("is_prod")):
        # 方案模拟里直接指定"每次加班小时"（并按每周加班天数限制）
        ot = float(scenario["ot_hours"])
    if scenario and scenario.get("ot_days_per_week") is not None:
        # 方案模拟：按周判断这天是否属于前 N 个加班日
        wk = (d.isocalendar()[0], d.isocalendar()[1])
        idx = _weekday_work_index(conn, d, cfg)
        if idx > int(scenario["ot_days_per_week"]):
            ot = 0.0

    if tc is not None:
        skilled += int(tc["borrow_in"] or 0) + int(tc["temp_add"] or 0)
        cut = int(tc["leave_cnt"] or 0) + int(tc["borrow_out"] or 0)
        take = min(skilled, cut)
        skilled -= take
        new -= max(0, cut - take)
    skilled, new = max(0, skilled), max(0, new)

    # 新人是否参与加班（默认不加班）
    new_ot = ot if bool(team.get("new_overtime", cfg["new_overtime"])) else 0.0
    hours = skilled * (float(team["sh_per_day"]) + ot) + new * (float(team["nh_per_day"]) + new_ot)
    return float(skilled + new), round(hours, 2), True


def _weekday_work_index(conn, d: dt.date, cfg: dict) -> int:
    """返回 d 是本周第几个工作日（周一=1），非工作日返回 99"""
    monday = d - dt.timedelta(days=d.weekday())
    idx = 0
    cur = monday
    while cur <= d:
        wd = cur.weekday()
        if (wd < 5) or (wd == 5 and cfg["work_sat"]) or (wd == 6 and cfg["work_sun"]):
            idx += 1
        cur += dt.timedelta(days=1)
    return idx


def build_capacity(conn, teams, start, days, scenario=None):
    cfg = get_worktime_config(conn)
    if scenario:
        for k in ("ot_days_per_week", "ot_hours", "work_sat", "work_sun", "new_overtime"):
            if k in scenario:
                cfg[k] = scenario[k]
    ensure_calendar(conn, start, days, (scenario or {}).get("extra_holidays"), cfg)
    end = start + dt.timedelta(days=days - 1)
    cal = {r["cal_date"]: dict(r) for r in conn.execute(
        "SELECT * FROM calendar WHERE cal_date BETWEEN ? AND ?",
        (start.isoformat(), end.isoformat()))}
    tcs = {(r["cal_date"], r["team"]): dict(r) for r in conn.execute(
        "SELECT * FROM team_calendar WHERE cal_date BETWEEN ? AND ?",
        (start.isoformat(), end.isoformat()))}
    table = {}
    for t in teams:
        d = start
        while d <= end:
            w, h, ok = team_capacity(conn, t, d, cal.get(d.isoformat()),
                                     tcs.get((d.isoformat(), t["team_code"])), scenario, cfg)
            table[(t["team_code"], d)] = (w, h, ok)
            d += dt.timedelta(days=1)
    return table


# ============================================================ 排程
class SchedTask:
    __slots__ = ("id", "ship_no", "ship_type", "code", "name", "team", "zone", "ttype",
                 "remaining", "status", "earliest_day", "min_w", "max_w", "critical",
                 "milestone", "preds", "prio", "insp", "earliest_date", "finish_date", "last_date")

    def __init__(self, row):
        self.id = row["id"]
        self.ship_no = row["ship_no"]
        self.ship_type = row["ship_type"] or ""
        self.code = row["process_code"]
        self.name = row["process_name"] or ""
        self.team = row["team"] or ""
        self.zone = row["zone"] or ""
        self.ttype = row["task_type"] or "通用"
        self.remaining = float(row["remaining_hours"] or 0)
        self.status = row["status"]
        self.earliest_day = int(row["earliest_day"] or 1)
        self.min_w = max(1, int(row["min_w"] or 1))
        self.max_w = max(1, int(row["max_w"] or 4))
        self.critical = bool(row["critical"])
        self.milestone = row["milestone"] or ""
        self.preds = []
        self.prio = 2
        self.insp = None
        self.earliest_date = None
        self.finish_date = None
        self.last_date = None


def _add_prod_days(start: dt.date, n: int, is_prod) -> dt.date:
    d, cnt, guard = start, 0, 0
    while cnt < n and guard < 3650:
        d += dt.timedelta(days=1)
        guard += 1
        if is_prod(d):
            cnt += 1
    return d


def _count_prod_days(start: dt.date, end: dt.date, is_prod) -> int:
    if not start or not end or end <= start:
        return 0
    n, d = 0, start
    while d < end:
        d += dt.timedelta(days=1)
        if is_prod(d):
            n += 1
    return n


# ============================================================ 船底启动门禁
# PMC 口径：船底没启动，后续装配工序根本无从开工 —— 整船不排产，也就不进班组周计划。
# 主计划表里"船底启动"有两列：实际启动日(sale_start) 与 计划启动日(plan_start)。
KEEL_RULES = ("auto", "plan", "sale")


def keel_status(ship, base_day, rule="auto"):
    """
    判断一艘船的"船底启动"是否满足排产的前置条件。

    返回 (可排产: bool, 判定说明: str, 生效的启动日: date|None)

    口径：
      * auto（默认）：实际启动日有值 -> 视为已启动；
                     否则要求"计划启动日 <= 排产基准日"，计划未到或没填都算未启动。
      * plan：只认计划启动日（没填或晚于基准日 = 未启动）。
      * sale：只认实际启动日（没填 = 未启动）。
    """
    ss = to_date(ship.get("sale_start"))
    ps = to_date(ship.get("plan_start"))

    if rule == "sale":
        if ss:
            return True, "实际船底启动 %s" % ss.isoformat(), ss
        return False, "未填实际船底启动日期", None

    if rule == "plan":
        if ps and ps <= base_day:
            return True, "计划船底启动 %s 已到" % ps.isoformat(), ps
        if ps:
            return False, "计划船底启动 %s 未到" % ps.isoformat(), ps
        return False, "未填计划船底启动日期", None

    # auto
    if ss:
        return True, "实际船底启动 %s" % ss.isoformat(), ss
    if ps and ps <= base_day:
        return True, "计划船底启动 %s 已到" % ps.isoformat(), ps
    if ps:
        return False, "计划船底启动 %s 未到" % ps.isoformat(), ps
    return False, "未填船底启动日期", None


def _run_schedule_impl(conn, start=None, horizon=120, freeze_days=0, scenario=None, note="", mode="normal",
                 keel_gate=None, keel_rule=None):
    # 两个独立维度：
    #   relaxed = 是否放开【单船并行人数上限 + 同舱区班组数上限】（仅理想口径放开）
    #             2026-10-15 用户确认：理想 = 放开这两条，但【单工序人数上限仍保留】。
    #   jit_on  = 是否用"准时制"排序（有余量的船让路，可单独用在标准产能上）
    relaxed = mode in ("ideal", "jit")
    jit_on = mode in ("jit", "normal_jit")
    """
    执行一次排程。返回 (version_no, metrics)
    """
    scenario = scenario or {}
    today = dt.date.today()
    start = start or today
    end = start + dt.timedelta(days=horizon - 1)

    # 船底启动门禁的开关与口径（可由调用方覆盖，否则读系统设置）
    if keel_gate is None:
        keel_gate = int(float(get_setting(conn, "keel_gate", 1) or 1))
    if keel_rule is None:
        keel_rule = (get_setting(conn, "keel_rule", "auto") or "auto").strip() or "auto"
    if keel_rule not in KEEL_RULES:
        keel_rule = "auto"

    teams = load_teams(conn)
    if not teams:
        return None, {"error": "没有班组配置"}
    team_map = {t["team_code"]: t for t in teams}
    team_codes = [t["team_code"] for t in sorted(teams, key=lambda x: TEAM_ORDER.get(x["team_code"], 99))]

    ships = [dict(r) for r in conn.execute("SELECT * FROM ships WHERE status='在制'")]
    ships = [s for s in ships if s["ship_no"] not in set(scenario.get("suspend_ships", []))]

    # ---- 船期延期（产能分析页表单写入 ship_delays）----
    # 只在本次排产里顺延该船的大节点与交检日，不改 ships 表原值（随时可撤销）。
    _delayed = []
    try:
        for r in conn.execute("SELECT ship_no,delay_days,reason FROM ship_delays "
                              "WHERE COALESCE(delay_days,0)<>0"):
            _d = float(r["delay_days"] or 0)
            for s in ships:
                if str(s["ship_no"]).strip().upper() != str(r["ship_no"]).strip().upper():
                    continue
                for _k in ("plan_start", "sale_start", "plan_demould", "sale_demould",
                           "plan_deck_bond", "plan_hardtop", "plan_inspection"):
                    _v = to_date(s.get(_k))
                    if _v:
                        s[_k] = (_v + dt.timedelta(days=_d)).isoformat()
                s["delay_days"] = _d
                _delayed.append({"ship_no": s["ship_no"], "delay_days": _d,
                                 "reason": r["reason"] or "",
                                 "new_inspection": s.get("plan_inspection")})
    except Exception as e:
        log("船期延期读取失败（忽略）：%s" % e)
    if _delayed:
        log("船期延期：%s" % "；".join(
            "%s 延%.0f天%s" % (x["ship_no"], x["delay_days"],
                                ("（%s）" % x["reason"]) if x["reason"] else "")
            for x in _delayed))

    # ---- 船底启动门禁 ----
    # 船底没启动，后续装配工序无从开工，整船不排产，也就不会进班组周计划。
    keel_blocked = []
    if keel_gate:
        _kept = []
        for s in ships:
            _ok, _why, _sd = keel_status(s, start, keel_rule)
            if _ok or (_sd and start < _sd <= end):
                _kept.append(s)
            else:
                keel_blocked.append({
                    "ship_no": s["ship_no"], "ship_type": s.get("ship_type") or "",
                    "plan_start": s.get("plan_start"), "sale_start": s.get("sale_start"),
                    "plan_inspection": s.get("plan_inspection"),
                    "remaining_hours": float(s.get("remaining_hours") or 0),
                    "reason": _why})
        if keel_blocked:
            _n_none = sum(1 for x in keel_blocked if not (x["plan_start"] or x["sale_start"]))
            log("船底启动门禁：排除 %d 艘（未填 %d / 未到 %d），口径=%s"
                % (len(keel_blocked), _n_none, len(keel_blocked) - _n_none, keel_rule))
        ships = _kept

    if not ships:
        if keel_blocked:
            return None, {"error": "在制船 %d 艘的船底均未启动/未到，无可排产船只"
                                   "（如需照排，可在【系统设置】关闭船底启动门禁）"
                                   % len(keel_blocked),
                          "keel_blocked": keel_blocked}
        return None, {"error": "没有在制船只"}
    ship_map = {s["ship_no"]: s for s in ships}
    ship_nos = [s["ship_no"] for s in ships]

    # ---- 优先级 ----
    for s in ships:
        pr = scenario.get("ship_priority", {}).get(s["ship_no"])
        if pr is None:
            pr = PRIORITY_MAP.get(s["manual_priority"] or "正常", 2)
            if s["must_finish_on_time"]:
                pr = max(0, pr - 1)
        s["priority"] = pr
        conn.execute("UPDATE ships SET priority=? WHERE ship_no=?", (pr, s["ship_no"]))
    conn.commit()

    # ---- 人力占用模式：这些船不按工序算工时，直接占用班组人力 ----
    mp_ships = {r["ship_no"] for r in conn.execute(
        "SELECT ship_no FROM ships WHERE manpower_mode=1")}
    # 班组 -> 船号 -> [(人数, 新人数, 开始日, 结束日)]（一船可有多段，按 seg 顺序）
    occupy = defaultdict(lambda: defaultdict(list))
    occupy_end = {}                     # 船号 -> 占用结束日
    for r in conn.execute("SELECT * FROM ship_manpower ORDER BY seg, id"):
        if r["ship_no"] not in mp_ships or int(r["workers"] or 0) <= 0:
            continue
        sd, ed = to_date(r["start_date"]), to_date(r["end_date"])
        occupy[r["team"]][r["ship_no"]].append(
            (int(r["workers"] or 0), int(r["new_workers"] or 0), sd, ed))
        if ed:
            occupy_end[r["ship_no"]] = max(occupy_end.get(r["ship_no"], ed), ed)
    if mp_ships:
        log(f"人力占用模式：{len(mp_ships)} 艘船直接占用人力，不参与工序排产")

    # ---- 待排工序（排除"人力占用"模式的船）----
    plan_ships = [s for s in ship_nos if s not in mp_ships]
    if not plan_ships:
        return None, {"error": "没有需要按工序排产的船（可能都设成了人力占用模式）"}
    q = "SELECT * FROM tasks WHERE ship_no IN (%s) AND remaining_hours>0 AND status<>?" % \
        ",".join("?" * len(plan_ships))
    rows = conn.execute(q, plan_ships + ["已完成"]).fetchall()
    upstream_rows = [r for r in rows if r['team'] in PRE_TEAMS]
    rows = [r for r in rows if r['team'] not in PRE_TEAMS]
    log(f"排程开始：{len(ships)} 艘在制船，待排工序 {len(rows)} 道，展望 {horizon} 天")

    deps = conn.execute("SELECT * FROM task_dependencies").fetchall()
    preds_map = defaultdict(list)
    for d in deps:
        preds_map[(d["ship_type"], d["process_code"])].append((d["predecessor_code"], d["lag_days"] or 0))

    # 冲突规则 -> (ta, tb, scope, level)
    # scope 缺省时按老字段 same_zone 推断：same_zone=1 -> zone，0 -> ship
    # （老口径 same_zone=0 = 全厂统计，是造成"A船吊装挡住B船"的原因，故收紧为 ship）
    conflicts = []
    for r in conn.execute("SELECT * FROM task_conflicts WHERE enabled=1"):
        try:
            _keys = r.keys()
        except AttributeError:
            _keys = []
        sc = r["scope"] if "scope" in _keys else None
        if not sc:
            sc = "zone" if r["same_zone"] else "ship"
        conflicts.append((r["ta"], r["tb"], sc, r["level"]))
    zone_cap = {r["zone_code"]: r["max_teams"] for r in conn.execute("SELECT * FROM zones")}

    capacity = build_capacity(conn, teams, start, horizon, scenario)

    # ---- 日历 is_prod ----
    cal_rows = conn.execute(
        "SELECT cal_date,is_prod FROM calendar WHERE cal_date BETWEEN ? AND ?",
        ((start - dt.timedelta(days=30)).isoformat(), (end + dt.timedelta(days=30)).isoformat())).fetchall()
    calmap = {r["cal_date"]: bool(r["is_prod"]) for r in cal_rows}
    extra_holi = {str(x)[:10] for x in scenario.get("extra_holidays", [])}

    def is_prod(d):
        if d.isoformat() in extra_holi:
            return False
        return calmap.get(d.isoformat(), d.weekday() < 6)

    # ---- 生产日锚点：把模板里"第 N 个生产日"翻译成真实日期 ----
    # 【为什么必须重算】旧逻辑直接拿 ships.plan_start 当锚点算第 N 天。
    # 但主计划里大多数船的船底启动日是 2024~2025 年、在制船也多是几个月前，
    # 于是"第 1 天"和"第 60 天"算出来的日期**全都落在过去**，
    # 约束相当于不存在 —— 排产会把这几十天的活全部堆在今天一起抢产能，
    # 生产周期表里排好的先后顺序被彻底打乱（PMC 反馈"逻辑上完全行不通"就是这个）。
    #
    # 现在的口径：
    #   · 船还没开工（plan_start >= 排产起始日）→ 锚点=plan_start，生产日 1 = 开工日
    #   · 船已开工 / 计划开工日已过期        → 锚点=排产起始日，
    #     把【最小的剩余生产日】对齐到今天，其余工序按与它的差值顺延
    #     （既保住生产周期的先后顺序和间隔，又不会排到过去）
    _rem_min = {}
    for r in rows:
        _k = r["ship_no"]
        _v = int(r["earliest_day"] or 1)
        if _k not in _rem_min or _v < _rem_min[_k]:
            _rem_min[_k] = _v
    ship_anchor = {}
    for _sno in ship_nos:
        _s = ship_map.get(_sno)
        if _s is None:
            continue
        _ps = to_date(_s["plan_start"])
        _dmin = _rem_min.get(_sno, 1)
        if _ps and _ps >= start:
            ship_anchor[_sno] = (_ps, 1)          # 未开工：按计划开工日正推
        else:
            ship_anchor[_sno] = (start, _dmin)    # 已开工：最小剩余生产日 = 今天
    _anch = [("%s 第%d天→%s" % (k, v[1], v[0].isoformat()))
             for k, v in sorted(ship_anchor.items())[:6]]
    if _anch:
        log("生产日锚点：" + "；".join(_anch) + ("…" if len(ship_anchor) > 6 else ""))

    # ---- 构建任务 ----
    tasks = []
    for r in rows:
        st = SchedTask(r)
        s = ship_map.get(st.ship_no)
        if s is None:
            continue
        st.preds = preds_map.get((st.ship_type, st.code), [])
        st.prio = s["priority"]
        st.insp = to_date(s["plan_inspection"])
        # 按生产日锚点算最早开工日：第 d 天 = 锚点 + (d - 锚点生产日) 个生产日
        _base, _base_day = ship_anchor.get(st.ship_no, (None, 1))
        if _base:
            _off = st.earliest_day - _base_day
            st.earliest_date = _base if _off <= 0 else _add_prod_days(_base, _off, is_prod)
        tm = team_map.get(st.team)
        if tm:
            # 单工序人数上限：2026-10-14 确认理想口径【也要保留】，
            # 不再区分 relaxed —— 两个口径都按 min(工序 max_w, 班组 max_per_task) 封顶。
            st.max_w = min(st.max_w, tm["max_per_task"])
        tasks.append(st)

    finished = {}
    for r in conn.execute("SELECT ship_no,process_code,actual_finish FROM tasks "
                          "WHERE ship_no IN (%s) AND status='已完成'" % ",".join("?" * len(ship_nos)),
                          ship_nos):
        finished[(r["ship_no"], r["process_code"])] = to_date(r["actual_finish"]) or (start - dt.timedelta(days=1))

    # 用户确认前置供给齐套，仅作为模型边界，不回写实际完工状态。
    for r in conn.execute("SELECT ship_no,process_code,team FROM tasks"):
        if r['ship_no'] in ship_map and r['team'] in PRE_TEAMS:
            finished[(r['ship_no'],r['process_code'])] = start - dt.timedelta(days=1)
    # ---- 冻结窗口 ----
    frozen = defaultdict(list)
    frozen_superseded = 0
    frozen_dropped = 0
    _keep_ships = set(ship_nos)          # 本次参与排产的船（剔除的船不在其中）
    if freeze_days > 0:
        ver = conn.execute("SELECT version_no FROM schedule_versions WHERE is_active=1 "
                           "ORDER BY id DESC LIMIT 1").fetchone()
        if ver:
            fr = conn.execute(
                "SELECT * FROM schedule_result WHERE version_no=? AND work_date BETWEEN ? AND ?",
                (ver["version_no"], start.isoformat(),
                 (start + dt.timedelta(days=min(freeze_days,horizon)-1)).isoformat())).fetchall()
            for r in fr:
                # 【已被剔除的船不再沿用冻结计划】
                # 上一版排了 A，但现在 A 已被移出排产（状态置为暂缓/冻结），
                # 它的冻结期计划不应再占用班组产能，否则"剔除"等于没生效。
                if r["ship_no"] not in _keep_ships or r["team"] in PRE_TEAMS:
                    frozen_dropped += 1
                    continue
                # 【已完成的任务不再作为未来待执行工作占用资源】
                # 上一版排了 A，但现场在 A 之前就把这道工序干完了 -> 冻结行不能锁死产能。
                # 判断依据：实际完工时间早于该冻结行日期（finished 里存的是 tasks.actual_finish）。
                _rf = r["work_date"]
                _af = finished.get((r["ship_no"], r["process_code"]))
                if _af is not None and to_date(_rf) is not None and _af <= to_date(_rf):
                    frozen_superseded += 1
                    continue
                frozen[to_date(r["work_date"])].append(
                    (r["ship_no"], r["process_code"], r["workers"] or 0,
                     r["planned_hours"] or 0, r["zone"] or "", r["team"]))
            _kept = sum(len(v) for v in frozen.values())
            if _kept:
                log(f"冻结窗口沿用 {_kept} 条既有计划（版本 {ver['version_no']}）")
            if frozen_superseded:
                log(f"冻结窗口剔除 {frozen_superseded} 条已被实际完工取代的旧计划")
            if frozen_dropped:
                log(f"冻结窗口剔除 {frozen_dropped} 条已不参与排产的船的计划")

    # 当天加班小时（人力占用也要按加班后的工时折算）
    cal_rows = {str(r["cal_date"])[:10]: dict(r) for r in conn.execute(
        "SELECT * FROM calendar WHERE cal_date BETWEEN ? AND ?",
        (start.isoformat(), end.isoformat()))}

    from precision_engine import allocate
    allocation = allocate(globals(), conn, tasks, finished, capacity, ships, teams,
                          start, end, frozen, conflicts, zone_cap, occupy, mp_ships, scenario, mode)
    active, results = allocation['active'], allocation['results']
    used_h, used_w = allocation['used_h'], allocation['used_w']
    starved_hours, queued_cnt = allocation['starved_hours'], allocation['queued_cnt']
    blocked, blocked_detail = allocation['blocked'], allocation['blocked_detail']

    # ---- 缺口分析：哪个时间段（按周）、哪个班组缺人、缺多少、要补多少人 ----
    gap_week = defaultdict(lambda: [0.0, 0.0, 0.0, 0])   # (周一,班组) -> [缺口H, 可用H, 已排H, 生产日数]
    d2 = start
    while d2 <= end:
        wk = d2 - dt.timedelta(days=d2.weekday())
        for tc in team_codes:
            c = capacity.get((tc, d2))
            cap_h = c[1] if (c and c[2]) else 0.0
            g = gap_week[(wk, tc)]
            g[1] += cap_h
            g[2] += used_h.get((tc, d2), 0.0)
            g[0] += starved_hours.get((d2, tc), 0.0)
            if cap_h > 0:
                g[3] += 1
        d2 += dt.timedelta(days=1)

    gap_rows = []
    for (wk, tc), v in sorted(gap_week.items()):
        demand, cap_h, used_h_, prod_days = v
        # 缺口 = 本周本可以干完的活 - 实际排掉的活（为负说明产能够用）
        gap_h = max(0.0, demand - used_h_)
        tinfo = next((t for t in teams if t["team_code"] == tc), {})
        per_day = float(tinfo.get("sh_per_day", 11.0))
        need_workers = int(math.ceil(gap_h / max(1.0, max(1, prod_days) * per_day))) if gap_h > 0.01 else 0
        load_rate = (used_h_ / cap_h * 100) if cap_h > 0 else 0.0
        full = 1 if load_rate >= 95 else 0
        # 区分"真缺人"和"被约束卡住"：只有产能真用满了，没排上的活才算人力缺口；
        # 否则是前后置/区域/单船人数上限卡着，加人也没用（不能算成缺口，否则数字虚高吓人）
        real_gap = gap_h if load_rate >= 85 else 0.0
        # 区分"真缺人"和"被约束卡住"：
        #   产能打满还有活排队 -> 确属人力不足
        #   产能没打满却有活排队 -> 是单船人数上限 / 区域互斥 / 前后置在起作用，加人也没用
        reason = ("产能已打满，确属人力不足" if full else
                  "负荷未满却有活排队：受单船人数上限/区域互斥/前后置约束，加人难以解决")
        gap_rows.append({
            "week_start": wk.isoformat(),
            "week_label": f"{wk.month}月{wk.day}日那周",
            "team": tc,
            "demand_hours": round(demand, 1),
            "gap_hours": round(gap_h, 1),
            "capacity": round(cap_h, 1),
            "planned": round(used_h_, 1),
            "load_rate": round(load_rate, 1),
            "full": full,
            "real_gap": round(real_gap, 1),
            "queued_tasks": queued_cnt.get((wk.isoformat(), tc), 0),
            "prod_days": prod_days,
            "need_workers": need_workers if full else 0,
            "reason": reason,
        })
    # 只保留真正有缺口的周
    gap_rows = [g for g in gap_rows if g["gap_hours"] > 0.01]

    # ---- 版本号 ----
    today_s = dt.date.today().strftime("%Y%m%d")
    cnt = conn.execute("SELECT COUNT(*) c FROM schedule_versions WHERE version_no LIKE ?",
                       (today_s + "%",)).fetchone()["c"]
    version = f"{today_s}_{cnt+1:03d}"

    conn.execute("UPDATE schedule_versions SET is_active=0")
    conn.execute("INSERT INTO schedule_versions(version_no,created_at,algorithm,params,metrics,is_active)"
                 " VALUES(?,?,?,?,?,1)",
                 (version, dt.datetime.now().isoformat(timespec="seconds"), "30分钟有限产能交期优先排程",
                  json.dumps({"start": start.isoformat(), "horizon": horizon,
                              "freeze": freeze_days, "scenario": scenario.get("name", "基准"),
                              # 口径：normal=标准(含舱室上人限制) /
                              # ideal=理想(放开单船人数+舱区班组数；单工序人数仍限4人)
                              #      jit=理想+JIT / normal_jit=标准+JIT
                              "mode": mode,
                              "keel_gate": keel_gate},
                             ensure_ascii=False), "{}", ))
    conn.commit()

    if results:
        # results 的第一位预留给版本号，这里补上（排程时版本号还没生成）
        rows = [(version,) + tuple(r[1:]) for r in results]
        step = 500
        for i in range(0, len(rows), step):
            conn.executemany(
                "INSERT INTO schedule_result(version_no,work_date,ship_no,ship_type,process_code,"
                "process_name,team,workers,planned_hours,zone,priority,reason)"
                " VALUES(?,?,?,?,?,?,?,?,?,?,?,?)", rows[i:i + step])
        conn.commit()

    for rid, times in zip(conn.execute('SELECT id FROM schedule_result WHERE version_no=? ORDER BY id', (version,)).fetchall(), allocation['slot_times']):
        conn.execute('UPDATE schedule_result SET planned_start=?,planned_end=? WHERE id=?', (times[0],times[1],rid[0]))
    # ---- 回写船只预测 ----
    # 先按"纯工时法"算一份预测作为兜底：
    # 展望期内没排完的船不能显示成空白，要给出外推的预测交检日
    est_pred, est_delay = {}, {}
    n_extrapolated = 0

    by_ship = defaultdict(list)
    for t in tasks:
        by_ship[t.ship_no].append(t)
    delay_total = 0
    delay_ships = []
    for s in ships:
        # 人力占用模式的船：预测完成 = 占用结束日（不按工序推算）
        if s["ship_no"] in mp_ships:
            pred = occupy_end.get(s["ship_no"])
            insp_d = to_date(s["plan_inspection"])
            delay = -1
            if pred and insp_d:
                delay = _count_prod_days(insp_d, pred, is_prod) if pred > insp_d else 0
            elif pred:
                delay = 0
            risk = ("高风险" if (delay < 0 or delay >= 5) else
                    ("风险" if delay >= 1 else "正常"))
            mp_rows = conn.execute(
                "SELECT team,workers,new_workers FROM ship_manpower WHERE ship_no=?",
                (s["ship_no"],)).fetchall()
            mp_desc = "、".join(f"{r['team']}{int(r['workers'] or 0)}人"
                                f"（新人{int(r['new_workers'] or 0)}）" for r in mp_rows)
            reason = (f"该船设为【人力占用模式】：不计算工序工时，直接占用 {mp_desc}"
                      + (f"，占用至 {pred}" if pred else ""))
            conn.execute("UPDATE ships SET predicted_finish=?,delay_days=?,risk_level=?,bottleneck=?,delay_reason=?,prediction_basis='人力占用结束日 非工序验证',feasibility_status='',unscheduled_hours=0 WHERE ship_no=?", (pred.isoformat() if pred else None,delay,'关注','',reason,s['ship_no']))
            continue

        ts = by_ship.get(s["ship_no"], [])
        finish_dates = [t.finish_date for t in ts if t.finish_date]
        # 【排程结果分级】（评审文档"四 区分排程验证结果和工时估算"）
        #   full        完整排程：所有工序都排进了展望期，用已排任务的最终完成时间
        #   partial     部分排程：排了一部分，剩下的用剩余工时外推 —— 只能算"估算"
        #   unscheduled 无法排程：一道都没排上（缺前置/被阻塞），预测仅供参考
        unsched_h = sum(max(0.0, t.remaining) for t in ts if not t.finish_date)
        if not ts:
            feasibility, basis = "", "无剩余工序"
            pred = None
        elif len(finish_dates) == len(ts):
            feasibility, basis = "full", "排程结果"
            pred = max(finish_dates)
        else:
            feasibility = "partial" if any(r[2]==s["ship_no"] for r in results) else "unscheduled"
            basis = "未排完 无可承诺交期"
            pred = None
        extrapolated = False
        if pred is None and ts:
            # 排程展望期内没排完 -> 用纯工时法外推，保证界面上始终看得到预测交检日
            if s["ship_no"] in est_pred:
                pred = est_pred[s["ship_no"]]
                extrapolated = True
                n_extrapolated += 1
                basis = "未排完 无可承诺交期"
        if pred and s["plan_inspection"]:
            insp = to_date(s["plan_inspection"])
            delay = _count_prod_days(insp, pred, is_prod) if pred > insp else 0
        elif not pred:
            delay = -1
        else:
            delay = 0
        if delay < 0:
            risk = "高风险"
        elif delay >= 5:
            risk = "高风险"
        elif delay >= 1:
            risk = "风险"
        elif delay > 0:
            risk = "关注"
        else:
            risk = "正常"
        if not s["plan_inspection"]:
            if risk == "正常": risk = "关注"
            basis += " 交检日期未维护"
        # 【排不完的船不能因为"日期没超"就报正常】
        # 交期判断必须同时看两件事：日期是否来得及 + 排程是否排得下。
        if risk == "正常" and feasibility in ("partial", "unscheduled"):
            risk = "关注"
        if delay > 0:
            delay_total += delay
            delay_ships.append(s["ship_no"])
        # 归因：先看"还剩多少活"，再看"排队等了几天"
        reason, bottleneck = "", ""
        if delay != 0:
            remain_by_team = defaultdict(float)
            cnt_by_team = defaultdict(int)
            for t in ts:
                if t.remaining > 1e-6:
                    remain_by_team[t.team] += t.remaining
                    cnt_by_team[t.team] += 1
            wait = sorted([(tm, c) for (sn, tm), c in blocked.items() if sn == s["ship_no"]],
                          key=lambda kv: -kv[1])
            wait_txt = "、".join(f"{tm} 排队等待 {c} 天" for tm, c in wait[:3])
            insp_d = to_date(s["plan_inspection"])
            overdue = (today - insp_d).days if insp_d else 0
            if pred and s["plan_inspection"]:
                head = f"计划交检 {s['plan_inspection']}，预测完成 {pred}，预计延期 {delay} 生产日。"
                if overdue > 0:
                    head += f"（计划交检日期已过期 {overdue} 个自然日，主计划日期可能未及时更新）"
            else:
                head = f"展望期内无法完成。"
                if overdue > 0:
                    head += f"（计划交检日期已过期 {overdue} 个自然日）"
            if remain_by_team:
                ranked = sorted(remain_by_team.items(), key=lambda kv: -kv[1])
                bottleneck = ranked[0][0]
                parts = "、".join(f"{tm} 剩余 {v:.0f}H/{cnt_by_team[tm]}道"
                                 for tm, v in ranked[:3])
                total_remain = sum(remain_by_team.values())
                reason = f"{head}剩余 {total_remain:.0f}H；班组缺口：{parts}"
                if wait_txt:
                    reason += f"；同期产能竞争：{wait_txt}"
            else:
                bottleneck = wait[0][0] if wait else ""
                reason = (f"{head}全部工序已排入计划，但受班组产能限制被推迟；"
                          f"主要等待：{wait_txt}" if wait_txt else
                          f"{head}受前后置 / 区域 / 工艺约束限制，无法进一步提前")
        if extrapolated:
            reason = (reason + "；" if reason else "") + \
                "【外推】排程展望期内未排完，预测交检按剩余工时推算"
        if feasibility == "partial":
            reason = (reason + "；" if reason else "") + \
                f"【部分排程】尚有 {unsched_h:.0f}H 未排入计划，不生成可承诺交期"
        elif feasibility == "unscheduled":
            reason = (reason + "；" if reason else "") + \
                f"【无法排程】{unsched_h:.0f}H 全部未排入，需先消除阻塞"
        conn.execute("UPDATE ships SET predicted_finish=?,delay_days=?,risk_level=?,"
                     "bottleneck=?,delay_reason=?,prediction_basis=?,feasibility_status=?,"
                     "unscheduled_hours=? WHERE ship_no=?",
                     (pred.isoformat() if pred else None, delay, risk, bottleneck,
                      reason, basis, feasibility, round(unsched_h, 2), s["ship_no"]))
    conn.commit()

    # ---- 班组负荷 ----
    load_rows = []
    for tc in team_codes:
        cap_h = used = 0.0
        peak = 0.0
        d2 = start
        while d2 <= end:
            c = capacity.get((tc, d2))
            ch = c[1] if (c and c[2]) else 0.0
            uh = used_h.get((tc, d2), 0.0)
            cap_h += ch
            used += uh
            if ch > 0:
                peak = max(peak, uh / ch * 100)
            d2 += dt.timedelta(days=1)
        load_rows.append({"team": tc, "capacity": round(cap_h, 1), "planned": round(used, 1),
                          "load_rate": round(used / cap_h * 100, 1) if cap_h else 0.0,
                          "peak_rate": round(peak, 1)})
    switches = 0
    dt_map = defaultdict(set)
    for r in results:
        dt_map[(r[1], r[6])].add(r[2])
    for k, v in dt_map.items():
        switches += max(0, len(v) - 1)

    for item in keel_blocked:
        conn.execute("UPDATE ships SET predicted_finish=NULL,delay_days=-1,risk_level='高风险',feasibility_status='unscheduled',prediction_basis='船底未启动',unscheduled_hours=(SELECT COALESCE(SUM(remaining_hours),0) FROM tasks WHERE ship_no=ships.ship_no AND status<>'已完成'),delay_reason=? WHERE ship_no=?",(item['reason'],item['ship_no']))
    # 未完成船数：既包括远期内做不完的（delay_days<0），
    # 也包括"还没排完"的（partial / unscheduled）——文档口径：按未排完任务计
    unfinished_ships = conn.execute(
        "SELECT COUNT(*) c FROM ships WHERE status='在制' AND (delay_days<0 "
        "OR feasibility_status IN ('partial','unscheduled'))").fetchone()["c"]
    feasi = {"full": 0, "partial": 0, "unscheduled": 0}
    for r in conn.execute("SELECT feasibility_status fs, COUNT(*) c FROM ships "
                          "WHERE status='在制' GROUP BY feasibility_status"):
        if r["fs"] in feasi:
            feasi[r["fs"]] = r["c"]
    metrics = {"upstream_assumption": "前置物料与大件已齐套 不排前置产能 不代表实际完工",
               "upstream_excluded_tasks": len(upstream_rows),
               "upstream_excluded_hours": round(sum(float(r['remaining_hours'] or 0) for r in upstream_rows),2),
               "engine": "时段排程 2026.10", "slot_minutes": 30,
               "planning_assumptions": "班组平均效率；08:00起班；12:00至13:00休息；尾段不足30分钟仍占整段；非实名人员排班",
               "unscheduled_detail": allocation['precision_details'],
               "team_load": load_rows, "switches": switches, "total_delay_days": delay_total,
               "delay_ships": delay_ships, "rows": len(results), "horizon": horizon,
               "unfinished": len(active), "unfinished_ships": unfinished_ships,
               "mode": mode,
               "extrapolated": n_extrapolated,
               "frozen_superseded": frozen_superseded,
               "frozen_dropped": frozen_dropped,
               "keel_blocked": keel_blocked[:200],
               "keel_blocked_count": len(keel_blocked),
               "keel_gate": int(keel_gate or 0),
               "keel_rule": keel_rule,
               "feasibility": feasi,
               "gap_by_week": gap_rows,
               "blocked_detail": blocked_detail[:300]}
    conn.execute("UPDATE schedule_versions SET metrics=? WHERE version_no=?",
                 (json.dumps(metrics, ensure_ascii=False), version))
    conn.commit()
    log(f"排程完成 版本 {version}：安排 {len(results)} 条，未完成 {len(active)} 道，"
        f"延期船 {len(delay_ships)} 艘")
    return version, metrics


# ============================================================ 五组综合人力建议
def suggest_headcount(conn, start=None, horizon=400, tolerance_days=0):
    """
    五个班组【一起】算：要满足主计划交检，各班组分别需要加多少人。

    方法（累计需求 vs 累计产能）：
      1. 每个班组，把它要服务的船按计划交检日从近到远排序
      2. 逐个累加"到这条船交检日为止必须干完的工时"= 累计需求
      3. 对应的"到那天为止能提供的工时"= 人数 × 每人每天工时 × 生产日数
      4. 取让【每一个时间点】都 产能 ≥ 需求 的最小人数 = 该班组所需人数
      5. 所需人数 - 现有人数 = 需要加的人数

    :param tolerance_days: 允许延期的天数（>0 会相应少加人）
    """
    start = start or dt.date.today()
    cfg = get_worktime_config(conn)
    ensure_calendar(conn, start, horizon, cfg=cfg)
    end = start + dt.timedelta(days=horizon)
    prod = [to_date(r["cal_date"]) for r in conn.execute(
        "SELECT cal_date FROM calendar WHERE cal_date BETWEEN ? AND ? AND is_prod=1 "
        "ORDER BY cal_date", (start.isoformat(), end.isoformat()))]
    prod_set = set(prod)

    def prod_days_to(d):
        if d <= start:
            return 1              # 交检日已过/就是今天：至少按 1 天算，避免除零
        n = sum(1 for x in prod if x <= d)
        if d > (prod[-1] if prod else start):
            n += (d - (prod[-1] if prod else start)).days
        return max(1, n)

    teams = load_teams(conn)
    # 每船每班组剩余工时
    rem = {}
    for r in conn.execute(
            "SELECT ship_no, team, SUM(remaining_hours) h FROM tasks "
            "WHERE status<>'已完成' AND remaining_hours>0 GROUP BY ship_no, team"):
        rem[(r["ship_no"], r["team"])] = float(r["h"] or 0)
    ships = {r["ship_no"]: dict(r) for r in conn.execute(
        "SELECT * FROM ships WHERE status='在制'")}

    out = []
    for t in teams:
        tc = t["team_code"]
        w, cap_h, ok = team_capacity(conn, t, start)
        per_day = (cap_h / w) if (w and ok and cap_h > 0) else float(t.get("sh_per_day", 11.0))
        if per_day <= 0:                      # 产能算成 0 时会除零，兜底
            per_day = float(t.get("sh_per_day", 11.0)) or 11.0
        cur_w = int(w or 0)
        # 该班组要服务的船
        jobs = []
        for (sno, t2), h in rem.items():
            if t2 != tc or h <= 0:
                continue
            s = ships.get(sno)
            if not s:
                continue
            insp = to_date(s["plan_inspection"])
            if insp is None:
                continue                      # 没有交检日的船不参与"按期"测算
            jobs.append((insp + dt.timedelta(days=int(tolerance_days)), h, sno))
        jobs.sort(key=lambda x: x[0])
        cum, need_peak, tight_ship, tight_date = 0.0, 0.0, "", ""
        for due, h, sno in jobs:
            cum += h
            days = prod_days_to(due)
            need = cum / (days * per_day) if per_day > 0 else 0
            if need > need_peak:
                need_peak, tight_ship, tight_date = need, sno, due.isoformat()
        need_w = int(math.ceil(need_peak)) if need_peak > 0 else cur_w
        add = max(0, need_w - cur_w)
        out.append({
            "team": tc, "workers": cur_w, "need_workers": need_w, "add": add,
            "per_day_hours": round(per_day, 2), "team_capacity": round(cap_h, 1),
            "total_remain": round(sum(h for _, h, _ in jobs), 1),
            "ships": len(jobs),
            "tight_ship": tight_ship, "tight_date": tight_date,
            "peak_need_workers": round(need_peak, 2),
        })
    return {"start": start.isoformat(), "tolerance_days": int(tolerance_days),
            "teams": out, "total_add": sum(x["add"] for x in out)}


def simulate_headcount_plan(conn, plan=None, start=None, horizon=120, freeze_days=0):
    """
    按给定的加人方案（可同时给五个班组加人）跑一次排程，看还剩几艘延期。
    plan: {"初装":2, "终装":5, ...}  表示各班组增加的人数
    """
    cur = conn.execute("SELECT version_no FROM schedule_versions WHERE is_active=1 "
                       "ORDER BY id DESC LIMIT 1").fetchone()
    base_version = cur["version_no"] if cur else None
    official_snap = _ship_snapshot(conn)     # 模拟前的正式结果，跑完要还原
    plan = plan or {}
    delta = {}
    for tc, n in plan.items():
        n = int(n or 0)
        if n > 0:
            delta[tc] = {"skilled": n, "new": 0}
    v, m = run_schedule(conn, start=start, horizon=horizon, freeze_days=freeze_days,
                        scenario={"team_delta": delta})
    snap = _ship_snapshot(conn)
    late = [s for s in snap.values() if s["delay"] > 0]
    res = {"version": v, "plan": plan, "delay_ships": len(late),
           "total_delay": sum(x["delay"] for x in snap.values() if x["delay"] > 0),
           "unfinished_ships": m.get("unfinished_ships", 0),
           "metrics": {"rows": m.get("rows", 0)}}
    _restore_official(conn, base_version, official_snap)
    return res


# ============================================================ 加人模拟 / 作息组合模拟
def _ship_snapshot(conn, ship_no=None):
    """抓取每条船的预测交检与延期天数"""
    rows = conn.execute(
        "SELECT ship_no,ship_type,plan_inspection,predicted_finish,delay_days,risk_level "
        "FROM ships WHERE status='在制'").fetchall()
    out = {}
    for r in rows:
        out[r["ship_no"]] = {
            "ship_type": r["ship_type"], "plan_inspection": r["plan_inspection"],
            "predicted": r["predicted_finish"], "delay": r["delay_days"] or 0,
            "risk": r["risk_level"],
        }
    return out


def _restore_official(conn, base_version, snap):
    """把模拟跑完后被改动的"正式结果"完整还原。

    背景（评审文档"五 方案隔离"）：
      run_schedule() 每跑一次都会 (1) 把新版本设为 is_active，
      (2) 回写 ships 表的 predicted_finish / delay_days / risk_level。
      模拟方案跑完后如果只把 is_active 改回原版本，船舶表的预测结果
      仍停留在模拟方案上 -> 正式排程与预测交期不一致（版本号没变、预测日期变了）。

    这里连同船舶表一起还原，保证"模拟不改正式结果"。
    """
    if base_version:
        conn.execute("UPDATE schedule_versions SET is_active=0")
        conn.execute("UPDATE schedule_versions SET is_active=1 WHERE version_no=?",
                     (base_version,))
    # 还原船舶表的预测结果（快照里有的船逐个还原）
    if snap:
        for s_no, s in snap.items():
            conn.execute("UPDATE ships SET predicted_finish=?, delay_days=?, risk_level=? "
                         "WHERE ship_no=?", (s.get("predicted"), s.get("delay") or 0,
                                             s.get("risk"), s_no))
    conn.commit()


def simulate_add_workers(conn, team, counts=(1, 2, 3, 5, 8, 10), new_workers=0, ship_no=None,
                         start=None, horizon=120, freeze_days=0, ship_max_override=0) -> dict:
    """
    对某个班组逐个增加 1 / 2 / 3 … 人，各跑一次排程，对比交检时间变化。

    直接回答："加 1 个、2 个、3 个人分别会怎样？加几个人能按期？"

    :param team:        要加人的班组（如 '终装'）
    :param counts:      依次模拟增加的人数
    :param new_workers: 加的是熟练工还是新人（0=熟练工）
    :param ship_no:     只关注某条船（为空则看总体）
    :return: {"base": {...}, "rows": [...]}
    """
    cur = conn.execute("SELECT version_no FROM schedule_versions WHERE is_active=1 "
                       "ORDER BY id DESC LIMIT 1").fetchone()
    base_version = cur["version_no"] if cur else None
    start = start or dt.date.today()
    # 【模拟前先记下正式结果】模拟跑完后要原样还原，否则船舶表的预测交检
    # 会永久停留在模拟方案上，与正式版本对不上（评审文档"五 方案隔离"）。
    official_snap = _ship_snapshot(conn)

    def _run(scen):
        v, m = run_schedule(conn, start=start, horizon=horizon,
                            freeze_days=freeze_days, scenario=scen)
        return v, m

    rows = []
    # 基准（不加人）单独跑一次，保证口径一致
    base_v, base_m = _run({})
    base_snap = _ship_snapshot(conn, ship_no)
    base_delay = sum(v["delay"] for v in base_snap.values() if v["delay"] > 0)
    base_unfinished = base_m.get("unfinished_ships", 0)

    for n in counts:
        scen = {"team_delta": {team: {"skilled": 0 if new_workers else n, "new": n if new_workers else 0}}}
        if ship_max_override:
            scen["ship_max_override"] = ship_max_override
        try:
            v, m = _run(scen)
            snap = _ship_snapshot(conn, ship_no)
            total_delay = sum(v2["delay"] for v2 in snap.values() if v2["delay"] > 0)
            s = snap.get(ship_no) if ship_no else None
            rows.append({
                "add": n,
                "team": team,
                "worker_type": "新人" if new_workers else "熟练工",
                "ship_no": ship_no or "(全部船)",
                "predicted": s["predicted"] if s else "",
                "delay": s["delay"] if s else "",
                "total_delay": total_delay,
                "delay_ships": len(m.get("delay_ships", [])),
                "unfinished_ships": m.get("unfinished_ships", 0),
                "version": v,
            })
        except Exception as e:
            log(f"加人模拟 +{n} 失败: {e}")
            rows.append({"add": n, "team": team, "error": str(e)})

    # 恢复基准版本为当前激活版本，并还原船舶表预测结果
    _restore_official(conn, base_version, official_snap)
    return {"base_version": base_v, "base_delay": base_delay,
            "base_unfinished": base_unfinished, "rows": rows,
            "target_ship": ship_no or "", "team": team}


def simulate_worktime_matrix(conn, ot_days_list=(0, 2, 3, 4, 5, 6, 7), sun_list=(0, 1),
                             ship_no=None, start=None, horizon=120, freeze_days=0) -> dict:
    """
    「每周加班 n 天 × 周日是否上班」组合矩阵，各跑一次排程。

    直接回答："一周加班几天才能做完？熟练工周日上班能不能做出来？"
    """
    cur = conn.execute("SELECT version_no FROM schedule_versions WHERE is_active=1 "
                       "ORDER BY id DESC LIMIT 1").fetchone()
    base_version = cur["version_no"] if cur else None
    start = start or dt.date.today()
    official_snap = _ship_snapshot(conn)     # 模拟前的正式结果，跑完要还原
    rows = []

    for sun in sun_list:
        for ot in ot_days_list:
            scen = {"ot_days_per_week": ot, "work_sun": sun}
            try:
                v, m = run_schedule(conn, start=start, horizon=horizon,
                                    freeze_days=freeze_days, scenario=scen)
                snap = _ship_snapshot(conn)
                total_delay = sum(x["delay"] for x in snap.values() if x["delay"] > 0)
                s = snap.get(ship_no) if ship_no else None
                rows.append({
                    "ot_days": ot, "work_sun": sun,
                    "desc": f"每周加班{ot}天" + ("+周日上班" if sun else ""),
                    "delay_ships": len(m.get("delay_ships", [])),
                    "total_delay": total_delay,
                    "unfinished_ships": m.get("unfinished_ships", 0),
                    "ship_no": ship_no or "",
                    "ship_predicted": s["predicted"] if s else "",
                    "ship_delay": s["delay"] if s else "",
                    "version": v,
                })
            except Exception as e:
                log(f"作息组合 加班{ot}天/周日{sun} 失败: {e}")
                rows.append({"ot_days": ot, "work_sun": sun, "error": str(e)})

    _restore_official(conn, base_version, official_snap)
    return {"rows": rows, "target_ship": ship_no or ""}


# ============================================================ 风险
def risk_rows(conn, version=None):
    """风险预警 —— 必须说清楚【哪个时间】【哪个环节】缺人、【缺多少工时】。

    每条风险都带 4 个可执行字段：
      时间   = 具体日期或日期区间（不是"近期"这种模糊话）
      缺工时H = 差多少人工时
      缺人数 = 折算成每天要补几个人
      建议   = 补人 / 加班 / 顺延哪条船
    """
    out = []
    today = dt.date.today()

    def add(kind, obj, level, desc, reason, advice, span="", lack_h=0.0, lack_w=0):
        out.append({"类型": kind, "对象": obj, "等级": level, "时间": span or "—",
                    "描述": desc, "原因": reason, "建议": advice,
                    "缺工时H": round(float(lack_h or 0), 1),
                    "缺人数": int(lack_w or 0)})

    # ---------- 1. 单船延期风险 ----------
    for r in conn.execute("SELECT * FROM ships WHERE status='在制'"):
        d = r["delay_days"] or 0
        if d < 0:
            add("船只风险", r["ship_no"], "高",
                "展望期内做不完，剩余 %.0fH 排不出去" % (r["remaining_hours"] or 0),
                r["delay_reason"] or "", "增加产能 / 暂缓低优先级船",
                "%s ~ %s" % (today.isoformat(), (today + dt.timedelta(days=120)).isoformat()),
                r["remaining_hours"] or 0)
        elif d > 0:
            add("船只风险", r["ship_no"], "高" if d >= 5 else "中",
                "预计延期 %d 生产日（计划交检 %s → 预测完成 %s）"
                % (d, r["plan_inspection"] or "未定", r["predicted_finish"] or "未定"),
                (r["delay_reason"] or "")[:150],
                "瓶颈班组：%s" % (r["bottleneck"] or "待分析"))
        if r["must_finish_on_time"] and d > 0:
            add("关键船", r["ship_no"], "高",
                "已设必须按期，仍预计延期 %d 生产日" % d,
                "优先级已提升，仍受人力/工艺约束", "暂缓低优先级船保该船")

    # ---------- 展望期与产能基线 ----------
    horizon_days = 120
    m = {}
    if version:
        row = conn.execute("SELECT params,metrics FROM schedule_versions WHERE version_no=?",
                           (version,)).fetchone()
        if row:
            try:
                horizon_days = int(json.loads(row["params"] or "{}").get("horizon") or 120)
            except Exception:
                pass
            try:
                m = json.loads(row["metrics"] or "{}")
            except Exception:
                m = {}
    span_dates = [r["cal_date"] for r in conn.execute(
        "SELECT cal_date FROM calendar WHERE cal_date>=? AND is_prod=1 ORDER BY cal_date LIMIT ?",
        (today.isoformat(), horizon_days))]
    span = ("%s ~ %s" % (span_dates[0], span_dates[-1])) if span_dates else            ("%s ~ %s" % (today.isoformat(), (today + dt.timedelta(days=horizon_days)).isoformat()))
    ndays = max(1, len(span_dates))
    remain_all = float(conn.execute(
        "SELECT COALESCE(SUM(remaining_hours),0) FROM tasks "
        "WHERE status<>'已完成' AND remaining_hours>0").fetchone()[0] or 0)
    teams = {t["team_code"]: t for t in load_teams(conn)}
    cal = {c["cal_date"]: dict(c) for c in conn.execute("SELECT * FROM calendar")}

    # ---------- 2. 班组产能缺口（哪个环节、缺多少工时、要几个人）----------
    for tc, t in teams.items():
        remain = float(conn.execute(
            "SELECT COALESCE(SUM(remaining_hours),0) FROM tasks "
            "WHERE team=? AND status<>'已完成' AND remaining_hours>0", (tc,)).fetchone()[0] or 0)
        if remain <= 0:
            continue
        cap_total = 0.0
        for ds in span_dates:
            _, h, _ok = team_capacity(conn, t, to_date(ds), cal.get(ds))
            cap_total += h
        cap_total = round(cap_total, 1)
        hpw = float(t.get("sh_per_day") or 11.0)     # 每人每天工时
        if cap_total <= 0 or remain > cap_total:
            lack = remain - max(0.0, cap_total)
            lack_w = max(1, int(round(lack / (ndays * hpw)))) if hpw else 1
            add("产能缺口", tc, "高",
                "%s 期间 %s 剩余 %.0fH，可用产能只有 %.0fH，缺 %.0fH"
                "（补 %d 人可消化，或整体顺延 %.1f 个生产日）"
                % (span, tc, remain, max(0.0, cap_total), lack,
                   lack_w, lack / max(1e-6, ndays * hpw)),
                "班组产能上限：熟练工%d+新人%d，日有效产能 %.1fH/人"
                % (t.get("skilled") or 0, t.get("new") or 0, hpw),
                "补 %d 人，或把 %.0fH 顺延到下一批" % (lack_w, lack) if lack_w
                else "加班 / 放宽大节点",
                span, lack, lack_w)

    # ---------- 3. 未排进展望期的工序（哪个环节卡住）----------
    det = m.get("unscheduled_detail") or []
    byteam = defaultdict(lambda: [0.0, 0])
    byship = defaultdict(lambda: [0.0, 0])
    for x in det:
        th = float(x.get("hours") or 0)
        byteam[x.get("team") or "?"][0] += th
        byteam[x.get("team") or "?"][1] += 1
        byship[x.get("ship_no") or "?"][0] += th
        byship[x.get("ship_no") or "?"][1] += 1
    for tc, (h, c) in byteam.items():
        t = teams.get(tc) or {}
        hpw = float(t.get("sh_per_day") or 11.0)
        lack_w = max(1, int(round(h / (ndays * hpw)))) if hpw else 1
        add("未排工序", tc, "高",
            "%d 道工序排不进 %s（%s），共 %.0fH 没人干" % (c, span, tc, h),
            "产能被前面的船占满，或大节点尚未到达",
            "补 %d 人 / 加班 / 顺延低优先级船（缺口只占展望期 %.1f%%）"
            % (lack_w, h / max(1e-6, remain_all) * 100), span, h, lack_w)
    for sn, (h, c) in list(byship.items())[:8]:
        add("未排工序(船)", sn, "中",
            "%s 有 %d 道工序排不进 %s，共 %.0fH" % (sn, c, span, h),
            "该船后续工序被产能或前后置卡住", "提高该船优先级 / 补人", span, h)

    # ---------- 4. 逐日超载（哪一天、哪个班组、超多少人）----------
    if version:
        agg = defaultdict(float)
        for r in conn.execute("SELECT work_date,team,planned_hours FROM schedule_result "
                              "WHERE version_no=?", (version,)):
            agg[(r["team"], r["work_date"])] += r["planned_hours"] or 0
        for (team, dstr), used in sorted(agg.items()):
            t = teams.get(team)
            if not t:
                continue
            d = to_date(dstr)
            _w, h, _ok = team_capacity(conn, t, d, cal.get(dstr))
            over = round(used - h, 1)
            if h > 0 and over > 0.05:          # 留 0.05H 容差，避免 100% 被浮点判成超载
                hpw = float(t.get("sh_per_day") or 11.0)
                add("人力超载", "%s / %s" % (team, dstr), "高",
                    "%s %s 安排 %.0fH，产能只有 %.0fH，超载 %.0fH（负荷率 %.0f%%）"
                    % (team, dstr, used, h, over, used / h * 100),
                    "当日工时超过班组有效产能", "加班 / 加人 / 把这 %.0fH 挪到后几天" % over,
                    dstr, over, int(round(over / hpw)) if hpw else 0)

    # ---------- 5. 数据缺失 ----------
    n_no_insp = conn.execute("SELECT COUNT(*) c FROM ships WHERE status='在制' AND "
                             "(plan_inspection IS NULL OR plan_inspection='')").fetchone()["c"]
    if n_no_insp:
        add("数据缺失", "主计划", "中",
            "%d 艘在制船没有计划交检日期，无法判断延期天数" % n_no_insp,
            "主计划未填写", "补齐交检日期")
    n_delay = conn.execute("SELECT COUNT(*) c FROM ship_delays "
                           "WHERE COALESCE(delay_days,0)<>0").fetchone()["c"] \
        if list(conn.execute("SELECT 1 FROM ship_delays LIMIT 1")) else 0
    if n_delay:
        rows = conn.execute("SELECT ship_no,delay_days FROM ship_delays "
                            "WHERE COALESCE(delay_days,0)<>0 ORDER BY delay_days DESC LIMIT 5").fetchall()
        add("人工延期", ", ".join("%s+%.0f天" % (r["ship_no"], r["delay_days"]) for r in rows),
            "中", "有 %d 艘船被人工设置了延期（已顺延大节点与交检日）" % n_delay,
            "产能分析页手工设置", "需要时在产能分析页把延期天数改回 0")
    return out


def suggest_actions(conn, ship_no):
    r = conn.execute("SELECT * FROM ships WHERE ship_no=?", (ship_no,)).fetchone()
    if not r:
        return []
    delay = r["delay_days"] or 0
    if delay <= 0:
        return [{"方案": "当前无需调整", "效果": "可按期完成", "代价": "-"}]
    out = []
    today = dt.date.today()
    end = to_date(r["plan_inspection"]) or (today + dt.timedelta(days=30))
    prod_days = max(1, int((end - today).days * 6 / 7))
    rows = conn.execute("SELECT team,SUM(remaining_hours) h,COUNT(*) c FROM tasks "
                        "WHERE ship_no=? AND remaining_hours>0 AND status<>'已完成' "
                        "GROUP BY team ORDER BY h DESC", (ship_no,)).fetchall()
    teams = {t["team_code"]: t for t in load_teams(conn)}
    for row in rows[:3]:
        team, hours = row["team"], row["h"] or 0
        t = teams.get(team, {})
        hpw = t.get("sh_per_day", 11.0)
        need = hours / max(1, prod_days * hpw)
        gap = max(0, int(round(need - 0.5)))
        if gap > 0:
            out.append({"方案": f"{team}增加 {gap} 人",
                        "效果": f"可消化 {team} 剩余 {hours:.0f}H（{prod_days} 个生产日内）",
                        "代价": f"人力 +{gap} 人"})
    out.append({"方案": "周日上班 + 每天加班 2H",
                "效果": f"全班组产能约 +25%，可缩短约 {max(1, int(delay*0.3))} 个生产日",
                "代价": "加班成本上升"})
    other = conn.execute("SELECT ship_no,remaining_hours FROM ships WHERE status='在制' AND "
                         "ship_no<>? ORDER BY priority DESC, remaining_hours DESC LIMIT 1",
                         (ship_no,)).fetchone()
    if other:
        out.append({"方案": f"暂缓 {other['ship_no']}",
                    "效果": f"释放 {other['remaining_hours'] or 0:.0f}H 产能给 {ship_no}",
                    "代价": f"{other['ship_no']} 可能延期"})
    out.append({"方案": f"{ship_no} 提到最高优先级", "效果": "优先占用各班组产能",
                "代价": "其他船顺延"})
    return out


# ============================================================ 工时核算对账明细
def audit_rows(conn, ship_no=None, team=None):
    """
    逐道工序的工时来源明细，用于和手工统计对账。

    每道工序给出：班组情况表的状态/工时、单船完成情况表的状态/剩余、
    选配项、系统最终采用的剩余工时、是否计入产能。
    """
    where, args = ["1=1"], []
    if ship_no:
        where.append("t.ship_no=?"); args.append(ship_no)
    if team:
        where.append("t.team=?"); args.append(team)
    q = """
    SELECT t.ship_no, t.ship_type, t.team, t.process_code, t.process_name,
           t.planned_hours, t.remaining_hours, t.status,
           (SELECT ap.status FROM actual_progress ap WHERE ap.ship_no=t.ship_no
              AND ap.process_code=t.process_code LIMIT 1) ap_status,
           (SELECT ap.hours FROM actual_progress ap WHERE ap.ship_no=t.ship_no
              AND ap.process_code=t.process_code LIMIT 1) ap_hours,
           (SELECT ap.option_flag FROM actual_progress ap WHERE ap.ship_no=t.ship_no
              AND ap.process_code=t.process_code LIMIT 1) ap_opt,
           (SELECT rp.raw_status FROM realtime_progress rp WHERE rp.ship_no=t.ship_no
              AND rp.process_code=t.process_code LIMIT 1) rt_status,
           (SELECT rp.remain_hours FROM realtime_progress rp WHERE rp.ship_no=t.ship_no
              AND rp.process_code=t.process_code LIMIT 1) rt_remain,
           (SELECT rp.quota_hours FROM realtime_progress rp WHERE rp.ship_no=t.ship_no
              AND rp.process_code=t.process_code LIMIT 1) rt_quota
    FROM tasks t WHERE %s ORDER BY t.ship_no, t.team, t.process_code""" % " AND ".join(where)
    out = []
    for r in conn.execute(q, args):
        d = dict(r)
        opt = str(d.get("ap_opt") or "").strip().lower()
        in_team = d.get("ap_status") is not None
        counted = (d["status"] != "已完成" and float(d["remaining_hours"] or 0) > 0)
        why = []
        if d["status"] == "已完成":
            why.append("已完成（不计入剩余）")
        elif float(d["remaining_hours"] or 0) <= 0:
            why.append("剩余工时为 0（已完工或不统计）")
        if not in_team:
            why.append("班组情况表未列（按规则不统计）")
        if opt in ("o", "0", "○", "〇"):
            why.append("选配O/0（不做时间统计）")
        d["in_team_plan"] = "有" if in_team else "无"
        d["option_flag"] = str(d.get("ap_opt") or "")
        d["counted"] = "计入" if counted else "不计入"
        d["reason"] = "；".join(why) or "未开始，按剩余工时计入"
        # 采用值是从哪来的（便于逐道核对"取小值"对不对）
        rem = float(d["remaining_hours"] or 0)
        ah, rr = d.get("ap_hours"), d.get("rt_remain")
        try:
            ah = float(ah) if ah is not None and float(ah) > 0 else None
        except (TypeError, ValueError):
            ah = None
        try:
            rr = float(rr) if rr is not None and float(rr) > 0 else None
        except (TypeError, ValueError):
            rr = None
        if ah and rr:
            same = abs(ah - rr) < 0.01
            d["source"] = ("两表一致" if same else
                           ("取班组表（较小）" if ah < rr else "取完成情况表（较小）"))
        elif ah:
            d["source"] = "仅班组表有"
        elif rr:
            d["source"] = "仅完成情况表有"
        else:
            d["source"] = "本船定额"
        out.append(d)
    return out


def export_audit(conn, path=None, ship_no=None, team=None):
    """导出工时核算对账明细 Excel"""
    rows = audit_rows(conn, ship_no, team)
    path = path or os.path.join(
        OUT_DIR, f"工时核算明细_{(ship_no or '全部船')}_{dt.date.today():%Y%m%d}.xlsx")
    head = ["船号", "船型", "班组", "工序编码", "工序名称", "系统定额H", "系统剩余H", "系统状态",
            "班组情况表", "班组表工时H", "选配项", "完成情况表状态", "完成情况表剩余H",
            "完成情况表定额H", "采用值来源", "是否计入", "判定说明"]
    data = [head]
    for r in rows:
        data.append([r["ship_no"], r["ship_type"], r["team"], r["process_code"],
                     r["process_name"], r["planned_hours"], r["remaining_hours"], r["status"],
                     r["in_team_plan"], r["ap_hours"], r["option_flag"],
                     r["rt_status"] or "-", r["rt_remain"], r["rt_quota"],
                     r["source"], r["counted"], r["reason"]])
    # 汇总
    s2 = [["按船×班组汇总"]]
    agg = {}
    for r in rows:
        if r["counted"] != "计入":
            continue
        k = (r["ship_no"], r["ship_type"], r["team"])
        agg[k] = agg.get(k, 0) + float(r["remaining_hours"] or 0)
    # 另算一版"纯班组表口径"：班组表里筛掉 已完成/已完工/X 和选配O/0，只看空白，
    # 直接把工时列求和 —— 这是 PMC 在 Excel 里手工筛选拿到的数，用于核对
    ref = {}
    try:
        # 不去重、直接把筛选后的行求和 —— 与在 Excel 里筛选后看求和值完全一致
        for r in conn.execute(
                "SELECT ship_no, team, SUM(hours) h FROM actual_progress "
                "WHERE (raw_status IS NULL OR raw_status='') "
                "  AND LOWER(COALESCE(option_flag,'')) NOT IN ('o','0') "
                "GROUP BY ship_no, team"):
            ref[(r["ship_no"], r["team"])] = round(float(r["h"] or 0), 1)
    except Exception:
        ref = {}
    s2 += [[], ["船号", "船型", "班组", "系统采用H（两表取小）",
                "班组表口径H（筛空白求和）", "差异"]]
    for (sno, st, tc), h in sorted(agg.items()):
        b = ref.get((sno, tc), 0.0)
        s2.append([sno, st, tc, round(h, 1), b, round(h - b, 1)])
    s2 += [[], ["说明：班组表口径 = 在班组情况表里筛掉 已完成/已完工/X 和选配O/0 后、" +
                "把工时列直接求和（对应你在 Excel 里手工筛选得到的数）"]]
    write_workbook(path, [("对账明细", data), ("按船班组汇总", s2)])
    log(f"已导出工时核算明细：{path}（{len(rows)} 道）")
    return path


# ============================================================ 交检预测（纯工时法）
def estimate_finish_by_hours(conn, teams=None, start=None, horizon=400, mode="queue"):
    """
    「只看剩余工时」的交检预测（不做工序级排程，也不看前后置/区域约束）。

    :param mode:
        'queue' = 标准模型：单船每天最多占用 min(班组日产能, 单船人数上限×人均工时)，
                  受"一条船最多站几个人"的现实限制；船按优先级排队，前面的先用产能。
        'ideal' = **理想模型**：不考虑上船区域/班组数限制、也不限单船人数，
                  一条船可动用班组的全部产能，只按时间测算（最快可能完成日）。

    算法：
      1. 每个班组算出每天的可用工时（熟练工×日工时 + 新人×日工时 + 当天加班）
      2. 单条船每天最多能占用 min(班组日产能, 单船人数上限 × 人均工时)
      3. 在制船按优先级排队，前面的船先用产能，用完才轮到下一艘
         需要天数 = ceil(该船剩余工时 ÷ 每天能占用的工时)
      4. 把"生产日"换算成真实日期（跳过周末、节假日）
      5. 一条船的最终预测交检 = 各班组的预测里最晚的那个

    :param teams: 要测算的班组列表；None / 空 = 五个班组全部
    :param mode:
        'queue' = 标准模型：单船每天最多占用 min(班组日产能, 单船人数上限×人均工时)，
                  也就是受"一条船最多站几个人"的现实限制；船按优先级排队，前面的先用产能。
        'ideal' = **理想模型**：不考虑上船区域限制、班组数限制、单船并行人数上限，
                  一条船可以用班组的全部产能，只按时间测算（最快可能完成日）。
                  注：**单工序人数上限（4 人）仍然保留**。
        'jit'   = **理想 + 准时制**：产能口径同 ideal，但船**按交检日紧迫度排序**
                  （交检早的排前面），卡在交检日完成 —— 与排产页「理想 + JIT」同口径。
        'normal_jit' = **标准 + 准时制**：产能口径同 queue（受单船人数上限约束），
                  但船按交检日紧迫度排序 —— 与排产页「标准 + JIT」同口径。

    :return: {"by_ship": [...], "by_team": {...}, "teams": [...]}
    """
    start = start or dt.date.today()
    cfg = get_worktime_config(conn)

    all_teams = load_teams(conn)
    team_codes = [t["team_code"] for t in all_teams]
    sel = [t for t in (teams or []) if t in TEAM_CODES] or list(TEAM_CODES)
    sel = [t for t in sel if t in team_codes]
    if not sel:
        sel = list(TEAM_CODES)

    # 产能口径：jit 与 ideal 一样放开单船人数限制；normal_jit 与 queue 一样受限
    relaxed = mode in ("ideal", "jit")
    jit_on = mode in ("jit", "normal_jit")

    # 生产日列表
    end = start + dt.timedelta(days=horizon)
    ensure_calendar(conn, start, horizon, cfg=cfg)
    prod_dates = [to_date(r["cal_date"]) for r in conn.execute(
        "SELECT cal_date FROM calendar WHERE cal_date BETWEEN ? AND ? AND is_prod=1 "
        "ORDER BY cal_date", (start.isoformat(), end.isoformat()))]
    if not prod_dates:
        prod_dates = [start]

    def prod_day_to_date(n):
        """第 n 个生产日（n 从 1 开始）对应的日期"""
        if n <= 0:
            return start
        if n <= len(prod_dates):
            return prod_dates[n - 1]
        return prod_dates[-1] + dt.timedelta(days=n - len(prod_dates))

    # 每条船在每个班组的剩余工时
    remain = defaultdict(float)
    for r in conn.execute(
            "SELECT ship_no, team, SUM(remaining_hours) h FROM tasks "
            "WHERE status<>'已完成' AND remaining_hours>0 GROUP BY ship_no, team"):
        remain[(r["ship_no"], r["team"])] = float(r["h"] or 0)

    ships = [dict(r) for r in conn.execute(
        "SELECT * FROM ships WHERE status='在制' ORDER BY priority, plan_inspection")]
    # 排队顺序：
    #   普通 / 理想模型 = 有交检日的排前面（交检早的优先），没有交检日的排后面
    #   准时制 JIT     = 按交检日紧迫度排（交检日早的先吃产能，卡着交检日做完），
    #                    这样"有余量的船"自然给"要交的船"让路 —— 与排产页 JIT 同口径。
    if jit_on:
        ships.sort(key=lambda s: (0 if s["plan_inspection"] else 1,
                                  str(s["plan_inspection"] or "9999")))
    else:
        ships.sort(key=lambda s: (0 if s["plan_inspection"] else 1,
                                  str(s["plan_inspection"] or "9999")))

    # 逐班组排队测算
    by_team = {}
    pred_by_ship = defaultdict(list)
    for tc in sel:
        tinfo = next((t for t in all_teams if t["team_code"] == tc), None)
        if not tinfo:
            continue
        w, cap_h, ok = team_capacity(conn, tinfo, start)
        hpw = (cap_h / w) if (w and ok) else float(tinfo.get("sh_per_day", 11.0))
        # 单船并行人数上限（初装6/机管4/电装4/甲板5/终装5）
        ship_cap = float(tinfo.get("ship_max", 5)) * hpw          # 单船每天最多占用
        if relaxed:
            # 理想 / 理想+JIT：放开单船人数上限，一条船可用班组全部产能
            ship_cap = cap_h if cap_h > 0 else ship_cap
        daily = min(cap_h, ship_cap) if cap_h > 0 else 0.0
        # 临时任务不在这里摊：下面排队是按天模拟的，会按当天实际占用扣除
        try:
            _tt = sum(float(r["hours"] or 0) for r in conn.execute(
                "SELECT hours FROM temp_tasks WHERE team=? AND COALESCE(enabled,1)=1", (tc,)))
        except Exception:
            _tt = 0.0
        cursor = 0
        rows = []
        for s in ships:
            rem = round(remain.get((s["ship_no"], tc), 0.0), 1)
            if rem <= 0:
                rows.append({"ship_no": s["ship_no"], "ship_type": s["ship_type"],
                             "remain": 0.0, "days": 0, "finish": "", "wait_days": cursor})
                continue
            # 逐日模拟（而不是用"总工时 ÷ 每天产能"整除）：
            # 这样才能把【临时任务】在某几天占掉的产能真实反映出来 —— 摊到整个窗口会稀释到看不出变化
            need = rem
            days = 0
            fd = prod_dates[-1] if prod_dates else None
            while need > 1e-6 and cursor < len(prod_dates):
                d = prod_dates[cursor]
                avail = daily - temp_task_daily_hours(conn, tc, d)
                # 理想 / 理想+JIT 放开单船人数：一条船可以用班组全部可用产能；
                # 标准口径每条船每天被 ship_cap 卡住。
                take = avail if relaxed else min(avail, ship_cap)
                if take > 0:
                    need -= take
                    fd = d
                cursor += 1
                days += 1
            pred_by_ship[s["ship_no"]].append(fd)
            rows.append({"ship_no": s["ship_no"], "ship_type": s["ship_type"],
                         "remain": rem, "days": days, "finish": fd.isoformat(),
                         "wait_days": cursor - days})
        by_team[tc] = {"daily_hours": round(daily, 1),
                       "team_capacity": round(cap_h, 1),
                       "temp_task_hours": round(_tt, 1),
                       "workers": w, "rows": rows,
                       "total_remain": round(sum(r["remain"] for r in rows), 1),
                       # 取最后一个"有预测日期"的船，而不是最后一行（最后一行可能不需要本班组）
                       "last_finish": next((r["finish"] for r in reversed(rows)
                                            if r.get("finish")), "")}

    # 每条船汇总
    out = []
    for s in ships:
        preds = pred_by_ship.get(s["ship_no"], [])
        pred = max(preds) if preds else None
        insp = to_date(s["plan_inspection"])
        delay = 0
        if pred and insp and pred > insp:
            prod_set = set(prod_dates)          # _count_prod_days 要的是判断函数，不是字典
            last = prod_dates[-1] if prod_dates else start
            delay = _count_prod_days(
                insp, pred,
                lambda d: (d in prod_set) or (d > last and d.weekday() < 5))
        per_team = {}
        for tc in sel:
            rr = next((x for x in by_team.get(tc, {}).get("rows", [])
                       if x["ship_no"] == s["ship_no"]), None)
            per_team[tc] = rr or {}
        out.append({
            "ship_no": s["ship_no"], "ship_type": s["ship_type"],
            "priority": s["manual_priority"] or "", "prio": s["priority"],
            "plan_inspection": str(s["plan_inspection"] or "")[:10],
            "predicted": pred.isoformat() if pred else "",
            "delay_days": delay,
            "risk": ("高风险" if (not pred or delay >= 5) else
                     ("风险" if delay >= 1 else "正常")),
            "teams": per_team,
        })
    out.sort(key=lambda x: (0 if x["plan_inspection"] else 1, x["plan_inspection"] or "9999"))

    return {"start": start.isoformat(), "teams": sel, "by_ship": out, "by_team": by_team,
            "prod_days": len(prod_dates), "mode": mode}


def export_estimate(conn, path=None, teams=None, start=None, mode="queue"):
    """导出交检预测（纯工时法）Excel"""
    r = estimate_finish_by_hours(conn, teams, start, mode=mode)
    sel = r["teams"]
    _mode_cn = {"queue": "标准模型", "ideal": "理想模型",
                "jit": "理想+JIT", "normal_jit": "标准+JIT"}
    path = path or os.path.join(OUT_DIR, f"交检预测_{_mode_cn.get(mode, mode)}_{dt.date.today():%Y%m%d}.xlsx")

    head = ["船号", "船型", "优先级", "计划交检", "预测交检", "延期(生产日)", "风险"]
    for tc in sel:
        head += [f"{tc}剩余H", f"{tc}需要天", f"{tc}预测"]
    rows = [head]
    for s in r["by_ship"]:
        line = [s["ship_no"], s["ship_type"], s["priority"], s["plan_inspection"],
                s["predicted"], s["delay_days"], s["risk"]]
        for tc in sel:
            t = s["teams"].get(tc) or {}
            line += [t.get("remain", 0), t.get("days", 0), t.get("finish", "")]
        rows.append(line)

    sheets = [("交检预测", rows)]
    # 每班组一张明细
    for tc in sel:
        d = r["by_team"].get(tc) or {}
        s2 = [[f"【{tc}】每日可用 {d.get('daily_hours', 0)} H（班组总产能 {d.get('team_capacity', 0)} H，"
               f"{d.get('workers', 0)} 人）"],
              [], ["船号", "船型", "剩余工时H", "需要天数", "等待天数", "预测完成"]]
        for x in d.get("rows", []):
            s2.append([x["ship_no"], x["ship_type"], x["remain"], x["days"],
                       x["wait_days"], x["finish"]])
        s2 += [[], ["合计剩余H", d.get("total_remain", 0)], ["最后完成", d.get("last_finish", "")]]
        sheets.append((f"{tc}测算", s2))
    write_workbook(path, sheets)
    log(f"已导出交检预测 Excel：{path}")
    return path


# ============================================================ 班组周 / 月计划
def team_plan(conn, version, team, mode="week", base_date=None,
              start=None, end=None):
    """
    生成某个班组的计划。

    :param mode: week=未来 7 天；month=下个月；range=自定义区间（配合 start/end）
    :param start/end: 自定义起止日期（几号到几号），给这两个参数时忽略 mode

    数据来源：
      * 排程结果 schedule_result（已考虑优先级、产能、前后置、区域）
      * 每道工序的完成情况（与实时进度表 / 班组周计划交叉核对的结果）
    """
    base = base_date or dt.date.today()
    if start and end:
        d0, d1 = start, end
        if d1 < d0:
            d0, d1 = d1, d0
        mode = "range"
    elif mode == "month":
        nm = (base.replace(day=28) + dt.timedelta(days=4)).replace(day=1)   # 下月 1 日
        d0 = nm
        d1 = ((nm.replace(day=28) + dt.timedelta(days=4)).replace(day=1)
              - dt.timedelta(days=1))                                       # 下月最后一天
    else:
        d0, d1 = base, base + dt.timedelta(days=6)

    rows = [dict(r) for r in conn.execute(
        "SELECT work_date,ship_no,ship_type,process_code,process_name,workers,"
        "planned_hours,zone,reason FROM schedule_result "
        "WHERE version_no=? AND team=? AND work_date BETWEEN ? AND ? "
        "AND process_code<>'MANPOWER' ORDER BY work_date, ship_no, process_code",
        (version, team, d0.isoformat(), d1.isoformat()))]

    # 附：每道工序当前的完成情况（交叉核对后的状态）
    prog = {}
    for r in conn.execute(
            "SELECT ship_no,process_code,status,progress,planned_hours,remaining_hours "
            "FROM tasks WHERE status='已完成'"):
        prog[(r["ship_no"], r["process_code"])] = 1.0
    for r in rows:
        d = to_date(r["work_date"])
        r["weekday"] = weekday_cn(d)
        r["done_before"] = prog.get((r["ship_no"], r["process_code"]), 0.0)

    # 按船汇总
    # 【用户口径】三源交替验证判定为已完工的工序，工时【不计入该船】。
    # 明细行仍保留并标注"该工序已完成=是"（方便与班组表对照），但不进任何汇总。
    by_ship = {}
    for r in rows:
        s = by_ship.setdefault(r["ship_no"], {"ship_no": r["ship_no"], "ship_type": r["ship_type"],
                                              "hours": 0.0, "days": set(), "tasks": 0,
                                              "done_hours": 0.0, "done_tasks": 0})
        if r["done_before"] >= 1.0:
            s["done_hours"] += float(r["planned_hours"] or 0)
            s["done_tasks"] += 1
            continue
        s["hours"] += float(r["planned_hours"] or 0)
        s["days"].add(r["work_date"])
        s["tasks"] += 1
    summary = []
    for s in by_ship.values():
        summary.append({"ship_no": s["ship_no"], "ship_type": s["ship_type"],
                        "hours": round(s["hours"], 1), "days": len(s["days"]),
                        "tasks": s["tasks"],
                        "first": min(s["days"]) if s["days"] else "",
                        "last": max(s["days"]) if s["days"] else "",
                        "done_hours": round(s["done_hours"], 1),
                        "done_tasks": s["done_tasks"]})
    summary.sort(key=lambda x: -x["hours"])
    # 整船全部工序都已完工 -> 该船不占产能，从汇总里去掉
    summary = [s for s in summary if s["hours"] > 0 or s["tasks"] > 0]

    # 按日汇总（周计划用）
    by_day = {}
    for r in rows:
        if r["done_before"] >= 1.0:
            continue
        by_day[r["work_date"]] = by_day.get(r["work_date"], 0.0) + float(r["planned_hours"] or 0)

    # 区间内每一天都列出来（含休息日），这样跨月/遇到假期时能看出"那天不生产"
    prod = {}
    try:
        for r in conn.execute(
                "SELECT cal_date, is_prod FROM calendar WHERE cal_date BETWEEN ? AND ?",
                (d0.isoformat(), d1.isoformat())):
            prod[r["cal_date"]] = int(r["is_prod"] or 0)
    except Exception:
        prod = {}
    all_days = []
    d = d0
    while d <= d1:
        k = d.isoformat()
        all_days.append({"date": k, "weekday": weekday_cn(d),
                         "is_prod": prod.get(k, 1 if d.weekday() < 5 else 0),
                         "hours": round(by_day.get(k, 0.0), 1)})
        d += dt.timedelta(days=1)
    n_rest = sum(1 for x in all_days if not x["is_prod"])
    return {"team": team, "mode": mode, "start": d0.isoformat(), "end": d1.isoformat(),
            "days": all_days, "rest_days": n_rest,
            "rows": rows, "summary": summary,
            "by_day": [{"date": k, "hours": round(v, 1)} for k, v in sorted(by_day.items())],
            "total_hours": round(sum(float(r["planned_hours"] or 0) for r in rows
                                     if r["done_before"] < 1.0), 1),
            "done_hours": round(sum(float(r["planned_hours"] or 0) for r in rows
                                    if r["done_before"] >= 1.0), 1)}


def export_team_plan(conn, version, mode="week", path=None, base_date=None):
    """
    导出各班组的周计划 / 月计划 Excel：五个班组各一张工作表。
    """
    label = "未来7天周计划" if mode == "week" else "下月月计划"
    base = base_date or dt.date.today()
    path = path or os.path.join(
        OUT_DIR, f"班组{'周' if mode == 'week' else '月'}计划_{version}.xlsx")

    short = "周计划" if mode == "week" else "月计划"
    sheets = []
    for team in TEAM_CODES:
        p = team_plan(conn, version, team, mode, base)
        rows = []
        # 明细
        rows.append([f"【{team}】{label}  {p['start']} ~ {p['end']}"])
        rows.append([])
        rows.append(["日期", "星期", "船号", "船型", "工序编码", "工序名称",
                     "人数", "工时H", "区域", "该工序已完成"])
        for r in p["rows"]:
            rows.append([r["work_date"], r["weekday"], r["ship_no"], r["ship_type"],
                         r["process_code"], r["process_name"], r["workers"],
                         round(float(r["planned_hours"] or 0), 1), r["zone"] or "",
                         "是" if r["done_before"] >= 1.0 else "否"])
        rows.append([])
        # 按船汇总
        rows.append(["—— 按船汇总（已完工工序的工时【不计入】） ——"])
        rows.append(["船号", "船型", "工序数", "投入工时H", "施工天数", "开始", "结束",
                     "其中已完工工序数", "已完工工时H(不计入)"])
        for s in p["summary"]:
            rows.append([s["ship_no"], s["ship_type"], s["tasks"], s["hours"],
                         s["days"], s["first"], s["last"],
                         s.get("done_tasks", 0), s.get("done_hours", 0)])
        rows.append([])
        rows.append(["合计投入工时H", p["total_hours"]])
        if p.get("done_hours"):
            rows.append(["其中已完工不计入H", p["done_hours"]])
        sheets.append((f"{team}·{short}", rows))

    # 加一张"班组人力占用"总览
    ov = [["班组", "计划投入工时H", "涉及船数"]]
    for team in TEAM_CODES:
        p = team_plan(conn, version, team, mode, base)
        ov.append([team, p["total_hours"], len(p["summary"])])
    sheets.append(("总览", ov))

    write_workbook(path, sheets)
    log(f"已导出班组{'周' if mode=='week' else '月'}计划 Excel：{path}")
    return path


# ============================================================ 导出
def export_plan(conn, version, path=None, extra=None):
    """
    导出排程报表（Excel，多工作表）。

    基础 5 张表：主计划预测 / 每日排产 / 班组日计划 / 人力负荷 / 风险预警
    附加表（传入 extra 时生成）：
      作息与加班设置 / 工作日历 / 进度交叉核对 / 产能缺口分析 / 加人模拟 / 加班组合模拟
    """
    path = path or os.path.join(OUT_DIR, f"排程计划_{version}.xlsx")
    extra = extra or {}

    ships = conn.execute("SELECT * FROM ships WHERE status='在制' "
                         "ORDER BY (plan_inspection IS NULL OR plan_inspection=''), "
                         "plan_inspection, priority").fetchall()
    s1 = [["船号", "船型", "优先级", "计划交检", "预测交检", "延期(生产日)", "风险等级",
           "排程完整性", "预测依据", "未排工时(H)", "完成比例", "剩余工时", "当前阶段",
           "主要瓶颈", "延期原因"]]
    _feasi_cn = {"full": "完整排程", "partial": "部分排程", "unscheduled": "无法排程"}
    for s in ships:
        s1.append([s["ship_no"], s["ship_type"],
                   {0: "紧急", 1: "高", 2: "正常", 3: "低", 4: "暂缓"}.get(s["priority"], s["manual_priority"]),
                   s["plan_inspection"] or "", s["predicted_finish"] or "",
                   s["delay_days"] if (s["delay_days"] or 0) >= 0 else "排不完",
                   s["risk_level"] or "",
                   _feasi_cn.get(s["feasibility_status"], s["feasibility_status"] or ""),
                   s["prediction_basis"] or "",
                   round(s["unscheduled_hours"] or 0, 1),
                   f"{(s['progress'] or 0)*100:.0f}%",
                   round(s["remaining_hours"] or 0, 1), s["current_stage"] or "",
                   s["bottleneck"] or "", (s["delay_reason"] or "")[:200]])

    # 同一天里也按【交检时间先后】排（交检早的在前），没有交检日的排最后
    res = conn.execute(
        "SELECT r.* FROM schedule_result r LEFT JOIN ships s ON s.ship_no=r.ship_no "
        "WHERE r.version_no=? ORDER BY r.work_date, "
        "(s.plan_inspection IS NULL OR s.plan_inspection=''), s.plan_inspection, "
        "r.ship_no, r.team", (version,)).fetchall()
    s2 = [["日期", "星期", "船号", "船型", "工序编码", "工序名称", "班组", "人数", "工时", "区域", "说明", "计划开始", "计划结束"]]
    for r in res:
        d = to_date(r["work_date"])
        s2.append([r["work_date"], weekday_cn(d), r["ship_no"], r["ship_type"], r["process_code"],
                   r["process_name"], r["team"], r["workers"], r["planned_hours"],
                   r["zone"] or "", (r["reason"] or "")[:80],r["planned_start"] or "",r["planned_end"] or ""])

    # 班组日计划
    s3 = [["班组", "日期", "船号", "工序编码", "工序名称", "人数", "工时", "区域"]]
    for team in [t["team_code"] for t in load_teams(conn)]:
        for r in res:
            if r["team"] == team:
                s3.append([team, r["work_date"], r["ship_no"], r["process_code"],
                           r["process_name"], r["workers"], r["planned_hours"], r["zone"] or ""])

    # 人力负荷
    teams = load_teams(conn)
    used = defaultdict(float)
    for r in res:
        used[(r["team"], r["work_date"])] += r["planned_hours"] or 0
    dates = sorted({r["work_date"] for r in res})
    s4 = [["日期", "星期", "班组", "可用人数", "可用H", "计划H", "剩余H", "负荷率", "超载H"]]
    for dstr in dates:
        d = to_date(dstr)
        for t in teams:
            w, h, ok = team_capacity(conn, t, d)
            u = used.get((t["team_code"], dstr), 0.0)
            s4.append([dstr, weekday_cn(d), t["team_code"], round(w, 1), round(h, 1), round(u, 1),
                       round(h - u, 1), f"{(u/h*100):.0f}%" if h else "-", round(max(0, u - h), 1)])

    s5 = [["类型", "对象", "等级", "描述", "原因", "建议"]]
    for r in risk_rows(conn, version):
        s5.append([r["类型"], r["对象"], r["等级"], r["描述"], r["原因"], r["建议"]])

    sheets = [("主计划预测", s1), ("每日排产", s2), ("班组日计划", s3),
              ("人力负荷", s4), ("风险预警", s5)]

    # ---------- 附加：作息与加班设置 ----------
    cfg = get_worktime_config(conn)
    s6 = [["项目", "当前值", "说明"]]
    s6 += [
        ["周六上班", "是" if cfg["work_sat"] else "否", "全局默认"],
        ["周日上班", "是" if cfg["work_sun"] else "否", "熟练工周日是否上班"],
        ["每周加班天数", cfg["ot_days_per_week"], "加班分配到每周的前 N 个工作日"],
        ["每次加班小时", cfg["ot_hours"], "熟练工每次加班时长"],
        ["新人参与加班", "是" if cfg["new_overtime"] else "否", "新人默认不加班"],
        ["熟练工日工时", cfg["sh_per_day"], "不含加班"],
        ["新人日工时", cfg["nh_per_day"], "不含加班"],
    ]
    s6.append([])
    s6.append(["班组", "熟练工", "新人", "熟练工日H", "新人日H", "周六", "周日",
               "每周加班天", "每次加班H", "新人加班", "单船上限", "赶工上限", "日产能H"])
    for t in load_teams(conn):
        s6.append([t["team_code"], t["skilled"], t["new_workers"], t["sh_per_day"],
                   t["nh_per_day"], "是" if t["work_sat"] else "否",
                   "是" if t["work_sun"] else "否", t["ot_days_per_week"], t["ot_hours"],
                   "是" if t["new_overtime"] else "否", t["ship_max"], t["ship_max_u"],
                   round(t["skilled"] * t["sh_per_day"] + t["new_workers"] * t["nh_per_day"], 1)])
    sheets.append(("作息与加班设置", s6))

    # ---------- 附加：工作日历（含加班日） ----------
    dates = sorted({r["work_date"] for r in res})
    s7 = [["日期", "星期", "类型", "是否生产日", "当日加班小时"]]
    for dstr in dates:
        d = to_date(dstr)
        r = conn.execute("SELECT * FROM calendar WHERE cal_date=?", (dstr,)).fetchone()
        s7.append([dstr, weekday_cn(d), r["day_type"] if r else "",
                   "是" if (r and r["is_prod"]) else "否", r["overtime"] if r else 0])
    sheets.append(("工作日历", s7))

    # ---------- 附加：进度交叉核对 ----------
    cross = extra.get("cross") or {}
    # 实时统计，不依赖导入时的快照
    try:
        conflict = conn.execute(
            "SELECT COUNT(*) FROM tasks t "
            "JOIN actual_progress ap ON ap.ship_no=t.ship_no AND ap.process_code=t.process_code "
            "JOIN realtime_progress rp ON rp.ship_no=t.ship_no AND rp.process_code=t.process_code "
            "WHERE ap.status='已完成' AND rp.done=0").fetchone()[0]
        opt_n, opt_h = conn.execute(
            "SELECT COUNT(*), ROUND(COALESCE(SUM(planned_hours),0),1) FROM tasks t "
            "JOIN actual_progress ap ON ap.ship_no=t.ship_no AND ap.process_code=t.process_code "
            "WHERE LOWER(TRIM(COALESCE(ap.option_flag,''))) IN ('o','0')").fetchone()
    except Exception:
        conflict, opt_n, opt_h = 0, 0, 0.0
    if not cross:
        # 导出时没有导入快照 -> 直接从库里统计，保证报表始终有数
        try:
            cross = {
                "from_team_plan": conn.execute(
                    "SELECT COUNT(*) FROM actual_progress WHERE status='已完成'").fetchone()[0],
                "from_realtime": conn.execute(
                    "SELECT COUNT(*) FROM realtime_progress WHERE done=1").fetchone()[0],
                "done": conn.execute(
                    "SELECT COUNT(*) FROM tasks WHERE status='已完成'").fetchone()[0],
            }
        except Exception:
            cross = {}
    s8 = [["核对项", "数量", "说明"]]
    s8 += [
        ["班组情况表报完成", cross.get("from_team_plan", "—"), "初装1组-总装1线 那类班组周计划"],
        ["单船完成情况表报完成", cross.get("from_realtime", "—"), "9-24 / 9-28 那类按船拉出的表"],
        ["交叉后判定完成", cross.get("done", "—"), "两个来源取并集：任一方完成即完成"],
        ["按完成情况表修正剩余", cross.get("remain_fixed", "—"), "用完成情况表的剩余工时覆盖"],
        ["完成情况表独有补建", cross.get("created", "—"), "模板中没有的工序"],
        ["★ 靠班组表补齐", conflict,
         "完成情况表说未完成、但班组情况表说已完成 -> 按已完成算"],
        ["★ 选配(O/0)不统计", opt_n,
         f"选配项=O/0 的工序不做时间统计（共 {opt_h} H），不占产能不排产"],
    ]
    s8.append([])
    s8.append(["船号", "工序编码", "工序名称", "班组", "班组表状态", "完成情况表状态",
               "选配项", "最终判定", "剩余H"])
    for r in conn.execute(
            "SELECT t.ship_no,t.process_code,t.process_name,t.team,t.status,t.remaining_hours,"
            "(SELECT ap.status FROM actual_progress ap WHERE ap.ship_no=t.ship_no AND "
            " ap.process_code=t.process_code LIMIT 1) ap_status,"
            "(SELECT ap2.option_flag FROM actual_progress ap2 WHERE ap2.ship_no=t.ship_no AND "
            " ap2.process_code=t.process_code LIMIT 1) opt,"
            "(SELECT rp.raw_status FROM realtime_progress rp WHERE rp.ship_no=t.ship_no AND "
            " rp.process_code=t.process_code LIMIT 1) rt_status "
            "FROM tasks t WHERE t.status='未完成' OR t.remaining_hours>0 "
            "ORDER BY t.ship_no LIMIT 400"):
        opt = r["opt"]
        s8.append([r["ship_no"], r["process_code"], r["process_name"], r["team"],
                   r["ap_status"] or "-", r["rt_status"] if r["rt_status"] is not None else "-",
                   (str(opt) if opt else "-"),
                   "未完成", round(r["remaining_hours"] or 0, 1)])
    sheets.append(("进度交叉核对", s8))

    # ---------- 附加：产能缺口分析 ----------
    gap = extra.get("gap")
    if not gap:
        v = conn.execute("SELECT metrics FROM schedule_versions WHERE version_no=?", (version,)).fetchone()
        if v and v["metrics"]:
            try:
                gap = json.loads(v["metrics"]).get("gap_by_week") or []
            except ValueError:
                gap = []
    if gap:
        s9 = [["周起始", "周标签", "班组", "缺口工时H", "当周可用H", "当周已排H",
               "生产日数", "建议补人数"]]
        for g in gap:
            s9.append([g["week_start"], g.get("week_label", ""), g["team"], g["gap_hours"],
                       g["capacity"], g["planned"], g.get("prod_days", 0), g.get("need_workers", 0)])
        sheets.append(("产能缺口分析", s9))

    # ---------- 附加：加人模拟 ----------
    wa = extra.get("whatif_add")
    if wa and wa.get("rows"):
        s10 = [["增加人数", "班组", "人员类型", "关注船", "预测交检", "延期天数",
                "延期船数", "总延期", "排不完船数"]]
        s10.append([0, wa.get("team", ""), "-", wa.get("target_ship") or "(全部)", "", "",
                    "", wa.get("base_delay", 0), wa.get("base_unfinished", 0)])
        for r in wa["rows"]:
            s10.append([r.get("add"), r.get("team"), r.get("worker_type", ""),
                        r.get("ship_no", ""), r.get("predicted", ""), r.get("delay", ""),
                        r.get("delay_ships", ""), r.get("total_delay", ""),
                        r.get("unfinished_ships", "")])
        sheets.append(("加人模拟", s10))

    # ---------- 附加：加班/周日组合模拟 ----------
    ww = extra.get("whatif_worktime")
    if ww and ww.get("rows"):
        s11 = [["每周加班天数", "周日上班", "组合说明", "延期船数", "总延期",
                "排不完船数", "关注船", "该船预测交检", "该船延期"]]
        for r in ww["rows"]:
            s11.append([r.get("ot_days"), "是" if r.get("work_sun") else "否",
                        r.get("desc", ""), r.get("delay_ships", ""), r.get("total_delay", ""),
                        r.get("unfinished_ships", ""), r.get("ship_no", ""),
                        r.get("ship_predicted", ""), r.get("ship_delay", "")])
        sheets.append(("加班组合模拟", s11))

    write_workbook(path, sheets)
    log(f"已导出 Excel（{len(sheets)} 个工作表）：{path}")
    return path

# ==========================================================================
#  主程序：本地服务 + 网页界面
# ==========================================================================
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
if BASE_DIR not in sys.path:
    sys.path.insert(0, BASE_DIR)


# ------------------------------------------------------------------
# 启动自检：任何一步出错都要"看得见"，绝不能黑框一闪而过
# ------------------------------------------------------------------
def _p(msg=""):
    """带 flush 的打印（避免输出被缓冲，导致窗口关闭后什么都没看到）"""
    try:
        print(msg, flush=True)
    except Exception:
        pass


def _wlog(msg):
    """同时写入启动日志，即使窗口已经关闭也能事后查看"""
    try:
        with open(os.path.join(BASE_DIR, "启动日志.txt"), "a", encoding="utf-8") as f:
            f.write(f"{dt.datetime.now():%Y-%m-%d %H:%M:%S}  {msg}\n")
    except Exception:
        pass


def _fatal(title, detail):
    """致命错误：打印 + 写日志 + 等用户按键（防止窗口闪退）"""
    _p("")
    _p("=" * 58)
    _p("  启动失败：" + title)
    _p("=" * 58)
    _p(detail)
    _p("")
    _p("详细信息也已写入：" + os.path.join(BASE_DIR, "启动日志.txt"))
    _p("")
    _wlog(f"[启动失败] {title}\n{detail}")
    try:
        input("按【回车键】关闭窗口…")
    except Exception:
        pass
    sys.exit(1)


_wlog("-" * 50)
_wlog(f"启动：Python {sys.version.split()[0]}  解释器 {sys.executable}")

DEFAULT_PORT = 8765
_log_lock = threading.Lock()


# ==================================================================
# API 处理
# ==================================================================
def _ships_rows(conn, statuses=None, keel_rule="auto"):
    """
    船只列表。

    statuses=None（默认）只返回 '在制' 船（历史行为，生产页面用）。
    传入 statuses 元组时按给定状态返回 —— 设置页传 ('在制','暂缓','冻结')
    以便把"已被剔除、不参与排产"的船也显示出来，方便随时放回。
    每行附加 in_plan：状态为 '在制' 才算本次参与排产。
    """
    today = dt.date.today().isoformat()
    if statuses:
        ph = ",".join("?" * len(statuses))
        sql = ("SELECT * FROM ships WHERE status IN (%s) "
               "ORDER BY CASE status WHEN '在制' THEN 0 WHEN '暂缓' THEN 1 ELSE 2 END, "
               "(plan_inspection IS NULL OR plan_inspection=''), plan_inspection, priority"
               % ph)
        args = list(statuses)
    else:
        sql = ("SELECT * FROM ships WHERE status='在制' "
               "ORDER BY (plan_inspection IS NULL OR plan_inspection=''), "
               "plan_inspection, priority")
        args = []
    out = []
    # 实时剩余工时的兜底：ships.remaining_hours 是 compute_ship_progress() 算出来的
    # "统计字段"，只有跑过 compute_ship_progress / 排产才会刷新。刚复制出来的船、
    # 或者从别处直接改过 tasks 的场景，这个字段可能是 0，界面上就会误导性地显示
    # "剩余 0H"（用户会以为复制失败）。所以这里一次性把各船的实时剩余算出来兜底。
    live_rem = {}
    live_hours = {}
    try:
        for r in conn.execute(
                "SELECT ship_no, "
                "SUM(CASE WHEN status<>'已完成' THEN remaining_hours ELSE 0 END) r, "
                "SUM(planned_hours) p, COUNT(*) n "
                "FROM tasks GROUP BY ship_no"):
            live_rem[str(r["ship_no"])] = float(r["r"] or 0)
            live_hours[str(r["ship_no"])] = (float(r["p"] or 0), int(r["n"] or 0))
    except Exception:
        pass
    for r in conn.execute(sql, args):
        d = dict(r)
        pi = str(d.get("plan_inspection") or "")[:10]
        # 交检日已过但仍在制 -> 单独标注（不退出排产）
        d["overdue"] = 1 if (pi and pi < today) else 0
        d["in_plan"] = 1 if (d.get("status") or "") == "在制" else 0
        # 船底启动状态（列表展示用，按当前判定口径）
        _kok, _kwhy, _kday = SC.keel_status(d, dt.date.today(), keel_rule)
        d["keel_ok"] = 1 if _kok else 0
        d["keel_reason"] = _kwhy
        # ---- 剩余工时：统计字段与实时值不一致时，用实时值，并标记出来 ----
        _sno = str(d.get("ship_no") or "")
        _stat = float(d.get("remaining_hours") or 0)
        _live = live_rem.get(_sno)
        _tot, _cnt = live_hours.get(_sno, (0.0, 0))
        d["task_count"] = _cnt
        d["total_hours"] = round(_tot, 1)
        if _live is not None and abs(_live - _stat) > 0.05:
            d["remaining_hours_stat"] = round(_stat, 1)   # 原始统计值（排查用）
            d["remaining_hours"] = round(_live, 1)        # 用实时值覆盖，界面不再显示 0
            d["remaining_stale"] = 1
        else:
            d["remaining_stale"] = 0
        out.append(d)
    return out


def api_status(h):
    conn = connect()
    try:
        files = IM.scan_raw_files()
        v = conn.execute("SELECT version_no FROM schedule_versions WHERE is_active=1 "
                         "ORDER BY id DESC LIMIT 1").fetchone()
        types_list = [dict(r) for r in conn.execute("SELECT * FROM ship_types")]

        def _names(val):
            """把扫描到的文件路径转成文件名列表；尚未导入时可能为 None"""
            if isinstance(val, str):
                return os.path.basename(val)
            return [os.path.basename(x) for x in (val or [])]

        data = {
            "today": dt.date.today().isoformat(),
            "version": v["version_no"] if v else "",
            "data_dir": DATA_DIR,
            "files": {k: _names(files[k]) for k in files},
            "ships": conn.execute("SELECT COUNT(*) c FROM ships WHERE status='在制'").fetchone()["c"],
            "tasks": conn.execute("SELECT COUNT(*) c FROM tasks").fetchone()["c"],
            "templates": conn.execute("SELECT COUNT(*) c FROM task_templates").fetchone()["c"],
            "types": len(types_list),
            "types_list": types_list,
            "progress": conn.execute("SELECT COUNT(*) c FROM actual_progress").fetchone()["c"],
            "quality": conn.execute("SELECT COUNT(*) c FROM data_quality").fetchone()["c"],
            "rows": conn.execute("SELECT COUNT(*) c FROM schedule_result").fetchone()["c"],
        }
        h.json(data)
    finally:
        conn.close()


def api_dashboard(h):
    conn = connect()
    try:
        ships = _ships_rows(conn)
        v = conn.execute("SELECT version_no,metrics FROM schedule_versions WHERE is_active=1 "
                         "ORDER BY id DESC LIMIT 1").fetchone()
        version = v["version_no"] if v else ""
        metrics = json.loads(v["metrics"]) if v and v["metrics"] else {}

        teams = SC.load_teams(conn)
        today = dt.date.today()
        used = defaultdict(float)
        if version:
            for r in conn.execute(
                    "SELECT team,SUM(planned_hours) h FROM schedule_result WHERE version_no=? "
                    "AND work_date BETWEEN ? AND ? GROUP BY team",
                    (version, today.isoformat(), (today + dt.timedelta(days=6)).isoformat())):
                used[r["team"]] = r["h"] or 0
        team_load = []
        for t in teams:
            cap = 0.0
            for i in range(7):
                w, hh, ok = SC.team_capacity(conn, t, today + dt.timedelta(days=i))
                if ok:
                    cap += hh
            u = used.get(t["team_code"], 0.0)
            team_load.append({"team": t["team_code"], "workers": t["skilled"] + t["new_workers"],
                              "capacity": round(cap, 1), "planned": round(u, 1),
                              "left": round(cap - u, 1),
                              "load_rate": (u / cap * 100) if cap else 0.0})

        on_time = len([s for s in ships if (s["delay_days"] or 0) == 0])
        risk = len([s for s in ships if s["risk_level"] in ("风险", "高风险")])
        delay = len([s for s in ships if (s["delay_days"] or 0) > 0])
        h.json({
            "version": version, "total": len(ships), "on_time": on_time, "risk": risk,
            "delay": delay, "delay_days": sum((s["delay_days"] or 0) for s in ships if (s["delay_days"] or 0) > 0),
            "remain": sum(s["remaining_hours"] or 0 for s in ships),
            "team_load": team_load, "ships": ships,
        })
    finally:
        conn.close()


def api_copy(h):
    copied = IM.copy_from_desktop()
    h.json({"copied": copied})


def api_import(h):
    conn = connect()
    try:
        init_db(conn)
        stats = IM.run_full_import(conn)
        with _log_lock:
            logs = list(aps_engine_logs())
        h.json({"stats": stats, "logs": logs})
    finally:
        conn.close()


def aps_engine_logs():
    return log_lines


def api_quality(h):
    conn = connect()
    try:
        rows = [dict(r) for r in conn.execute(
            "SELECT * FROM data_quality ORDER BY severity, src, row_no LIMIT 400")]
        h.json(rows)
    finally:
        conn.close()


def _num_param(body, key, default, lo=None, hi=None):
    """取数值参数：区分"没传"和"显式传 0"。
    旧写法 int(body.get("freeze") or 2) 会把 0 当成假值替换成 2，
    导致用户把冻结天数设为 0 也不生效。
    """
    if body is None or key not in body or body.get(key) is None or body.get(key) == "":
        v = default
    else:
        try:
            v = int(float(body.get(key)))
        except (TypeError, ValueError):
            v = default
    if lo is not None:
        v = max(lo, v)
    if hi is not None:
        v = min(hi, v)
    return v


def api_schedule(h, body):
    conn = connect()
    try:
        init_db(conn)
        start = to_date(body.get("start")) or dt.date.today()
        horizon = _num_param(body, "horizon", 120, lo=1, hi=730)
        freeze = _num_param(body, "freeze", 0, lo=0, hi=90)   # 默认 0：冻结要主动填，别默认 2 让人漏看
        mode = (body or {}).get("mode") or "normal"
        if mode not in ("normal", "ideal", "jit", "normal_jit"):
            mode = "normal"
        # 「本次无视船底门禁」：调用方显式传 keel_gate=0 时，只对本次排产生效
        _kg = (body or {}).get("keel_gate")
        keel_gate = None if _kg is None else (1 if int(_kg or 0) else 0)
        version, metrics = SC.run_schedule(conn, start=start, horizon=horizon, freeze_days=freeze,
                                          mode=mode, keel_gate=keel_gate)
        if version is None:
            h.json({"error": metrics.get("error", "排产失败")})
            return
        h.json({"version": version, "metrics": metrics})
    finally:
        conn.close()


def api_ships(h):
    conn = connect()
    try:
        v = conn.execute("SELECT version_no,metrics FROM schedule_versions WHERE is_active=1 "
                         "ORDER BY id DESC LIMIT 1").fetchone()
        version = v["version_no"] if v else ""
        metrics = json.loads(v["metrics"]) if v and v["metrics"] else {}
        h.json({"version": version, "ships": _ships_rows(conn),
                "rows": metrics.get("rows", 0), "unfinished": metrics.get("unfinished", 0),
                "delay_ships": metrics.get("delay_ships", []),
                "total_delay": metrics.get("total_delay_days", 0)})
    finally:
        conn.close()


def api_gantt(h, q):
    conn = connect()
    try:
        v = conn.execute("SELECT version_no FROM schedule_versions WHERE is_active=1 "
                         "ORDER BY id DESC LIMIT 1").fetchone()
        if not v:
            h.json({"dates": [], "rows": []})
            return
        mode = q.get("mode", ["ship"])[0]
        key = q.get("key", [""])[0]
        days = int(q.get("days", [60])[0])
        where = "ship_no=?" if mode == "ship" else "team=?"
        rows = conn.execute(
            f"SELECT * FROM schedule_result WHERE version_no=? AND {where} "
            f"ORDER BY work_date", (v["version_no"], key)).fetchall()
        if not rows:
            h.json({"dates": [], "rows": []})
            return
        start = to_date(rows[0]["work_date"])
        end = start + dt.timedelta(days=days - 1)
        dates = []
        d = start
        while d <= end:
            dates.append({"d": f"{d.month}/{d.day}", "full": d.isoformat(), "wk": d.weekday()})
            d += dt.timedelta(days=1)
        idx = {d["full"]: i for i, d in enumerate(dates)}

        by_key = defaultdict(lambda: [None] * len(dates))
        labels = {}
        for r in rows:
            dstr = r["work_date"]
            if dstr not in idx:
                continue
            label = (f"{r['process_code']} {r['process_name']}"[:26] if mode == "ship"
                     else r["ship_no"])
            labels[label] = 1
            by_key[label][idx[dstr]] = {"team": r["team"], "w": r["workers"],
                                        "h": r["planned_hours"], "zone": r["zone"],
                                        "name": r["process_name"]}
        out = [{"label": k, "cells": v} for k, v in by_key.items()]
        order = {t: i for i, t in enumerate(TEAM_CODES)}
        out.sort(key=lambda x: (order.get(x["cells"][[i for i, c in enumerate(x["cells"]) if c][0]]["team"], 99)
                                if any(x["cells"]) else 99, x["label"]))
        h.json({"dates": dates, "rows": out})
    finally:
        conn.close()


def api_capacity(h, q):
    conn = connect()
    try:
        start = to_date(q.get("start", [None])[0]) or dt.date.today()
        days = int(q.get("days", [30])[0])
        teams = SC.load_teams(conn)
        v = conn.execute("SELECT version_no FROM schedule_versions WHERE is_active=1 "
                         "ORDER BY id DESC LIMIT 1").fetchone()
        used = defaultdict(float)
        if v:
            for r in conn.execute(
                    "SELECT team,work_date,SUM(planned_hours) h FROM schedule_result "
                    "WHERE version_no=? GROUP BY team,work_date", (v["version_no"],)):
                used[(r["team"], r["work_date"])] = r["h"] or 0
        cap = SC.build_capacity(conn, teams, start, days)
        dates, rows = [], []
        d = start
        while d <= (start + dt.timedelta(days=days - 1)):
            dates.append(d.isoformat())
            for t in teams:
                w, hh, ok = cap.get((t["team_code"], d), (0, 0, False))
                u = used.get((t["team_code"], d.isoformat()), 0.0)
                rows.append({"date": d.isoformat(), "wk": weekday_cn(d), "team": t["team_code"],
                             "prod": "是" if ok else "否", "workers": round(w, 1),
                             "capacity": round(hh, 1), "planned": round(u, 1),
                             "left": round(hh - u, 1), "rate": (u / hh * 100) if hh else 0.0,
                             "over": max(0.0, u - hh)})
            d += dt.timedelta(days=1)
        summary = []
        for t in teams:
            sub = [r for r in rows if r["team"] == t["team_code"]]
            c = sum(r["capacity"] for r in sub)
            u = sum(r["planned"] for r in sub)
            summary.append({"team": t["team_code"], "capacity": round(c, 1), "planned": round(u, 1),
                            "left": round(c - u, 1), "load_rate": (u / c * 100) if c else 0.0,
                            "over_days": len([r for r in sub if r["over"] > 0])})
        h.json({"dates": dates, "rows": rows, "summary": summary})
    finally:
        conn.close()


def api_risk(h):
    conn = connect()
    try:
        v = conn.execute("SELECT version_no FROM schedule_versions WHERE is_active=1 "
                         "ORDER BY id DESC LIMIT 1").fetchone()
        ships = [r["ship_no"] for r in conn.execute(
            "SELECT ship_no FROM ships WHERE status='在制' ORDER BY priority")]
        h.json({"risks": SC.risk_rows(conn, v["version_no"] if v else None), "ships": ships})
    finally:
        conn.close()


def api_ship_delay(h, body):
    """船期延期设置（产能分析页表单）
       POST {act:'list'}                       取全部在制船 + 当前延期
       POST {act:'set', ships:[...], delay_days:N, reason:'...'}   批量设置
       POST {act:'clear', ships:[...]}         延期清零
    """
    conn = connect()
    try:
        init_db(conn)
        body = body or {}
        act = body.get("act") or "list"
        if act == "list":
            delays = {}
            try:
                for r in conn.execute("SELECT * FROM ship_delays"):
                    delays[str(r["ship_no"]).strip().upper()] = dict(r)
            except Exception:
                pass
            out = []
            for r in conn.execute(
                    "SELECT ship_no,ship_type,plan_inspection,predicted_finish,remaining_hours,"
                    "delay_days,bottleneck FROM ships WHERE status='在制' "
                    "ORDER BY ship_no"):
                sn = str(r["ship_no"]).strip().upper()
                d = delays.get(sn) or {}
                out.append({
                    "ship_no": sn, "ship_type": r["ship_type"] or "",
                    "plan_inspection": r["plan_inspection"] or "",
                    "predicted_finish": r["predicted_finish"] or "",
                    "remaining_hours": r["remaining_hours"] or 0,
                    "bottleneck": r["bottleneck"] or "",
                    "delay_days": d.get("delay_days") or 0,
                    "reason": d.get("reason") or "",
                    "updated_at": d.get("updated_at") or "",
                    # 顺延后的预计交检日，方便确认效果
                    "delayed_inspection": _shift_date(
                        r["plan_inspection"], float(d.get("delay_days") or 0))})
            return h.json({"ok": True, "rows": out})

        ships = body.get("ships") or []
        if isinstance(ships, str):
            ships = [x for x in ships.replace("；", ",").replace(";", ",").split(",")]
        ships = [str(x).strip().upper() for x in ships if str(x).strip()]
        if not ships:
            return h.json({"error": "请先勾选要设置延期的船"})
        if act == "clear":
            n = 0
            for sn in ships:
                n += conn.execute("DELETE FROM ship_delays WHERE UPPER(TRIM(ship_no))=?",
                                  (sn,)).rowcount
            conn.commit()
            return h.json({"ok": True, "cleared": n})

        try:
            days = float(body.get("delay_days") or 0)
        except (TypeError, ValueError):
            return h.json({"error": "延期天数必须是数字"})
        if days < -365 or days > 3650:
            return h.json({"error": "延期天数超出范围（-365 ~ 3650）"})
        reason = str(body.get("reason") or "").strip()[:200]
        now = dt.datetime.now().isoformat(timespec="seconds")
        saved, missed = [], []
        for sn in ships:
            real = _ship_exists(conn, sn)
            if real is None:
                missed.append(sn)
                continue
            conn.execute("INSERT OR REPLACE INTO ship_delays(ship_no,delay_days,reason,updated_at) "
                         "VALUES(?,?,?,?)",
                         (real, days, reason, now))
            saved.append(real)
        conn.commit()
        log("船期延期：%s 设为 %.0f 天%s"
            % ("、".join(saved), days, ("（%s）" % reason) if reason else ""))
        h.json({"ok": True, "saved": saved, "missed": missed, "delay_days": days})
    finally:
        conn.close()


def _shift_date(dstr, days):
    if not dstr or not days:
        return dstr or ""
    d = to_date(dstr)
    return (d + dt.timedelta(days=int(round(days)))).isoformat() if d else (dstr or "")


def api_excluded_ships(h, body=None):
    """作废船黑名单：查看 / 恢复
       POST {act:'list'}                  列出已作废的船
       POST {act:'restore', ships:[...]}  恢复（会自动从历史备份捞回工序与报工）
       POST {act:'block',  ships:[...]}   直接作废（不删工序，仅拉黑）
    """
    conn = connect()
    try:
        init_db(conn)
        body = body or {}
        act = body.get("act") or "list"
        if act == "list":
            return h.json({"ok": True, "rows": list_excluded(conn),
                           "backups": _list_backups()})
        ships = body.get("ships") or []
        if isinstance(ships, str):
            ships = [x for x in ships.replace("；", ",").replace(";", ",").split(",")]
        ships = [str(x).strip().upper() for x in ships if str(x).strip()]
        if not ships:
            return h.json({"error": "请先勾选船号"})
        if act == "restore":
            # 先查船还在不在：不在的话要从历史备份整条捞回来
            out = []
            for sn in ships:
                real = _ship_exists(conn, sn)
                if real is None:
                    r = _restore_ship_from_backup(conn, sn)
                    out.append({"ship_no": sn, "ok": r.get("ok"),
                                "detail": r.get("detail", "")})
                else:
                    out.append({"ship_no": real, "ok": True,
                                "detail": "船仍在库中，数据未变"})
                # 无论走哪条路径，都必须移出作废黑名单 ——
                # 否则数据回来了却仍被一键导入当作"已作废"跳过
                if restore_ship(conn, sn):
                    out[-1]["detail"] += "；已移出作废名单"
            conn.commit()
            bad = [x for x in out if not x["ok"]]
            return h.json({"ok": not bad, "restored": out,
                           "error": None if not bad else
                           "；".join("%s：%s" % (x["ship_no"], x["detail"]) for x in bad)})
        if act == "block":
            reason = str(body.get("reason") or "界面作废")[:120]
            with_tasks = 1 if body.get("with_tasks") else 0
            done = []
            for sn in ships:
                r = delete_ship(conn, sn, with_tasks=bool(with_tasks), block=True)
                if r.get("ok"):
                    done.append(sn)
            conn.commit()
            return h.json({"ok": True, "blocked": done})
        h.json({"error": "未知 act=%s" % act})
    finally:
        conn.close()


def _list_backups():
    """data 目录下的历史备份列表（新→旧），供界面显示"可从哪恢复" """
    try:
        import glob as _g
        ps = [p for p in _g.glob(os.path.join(DATA_DIR, "aps*.db"))
              if os.path.basename(p) != "aps.db"]
        return [os.path.basename(p) for p in
                sorted(ps, key=os.path.getmtime, reverse=True)][:8]
    except Exception:
        return []


def _open_ro(path):
    """只读打开一个 sqlite 库。

    ⚠ URI 必须用 pathlib 的 as_uri() 生成 `file:///C:/...`。
    手写 `file:C:/...` 会被 SQLite 当成**相对路径**（"C:" 开头的目录名），
    于是"打不开"却不一定报错 —— 表现就是"明明有备份却找不到"。
    """
    from pathlib import Path as _P
    uri = _P(os.path.abspath(path)).as_uri() + "?mode=ro"
    return sqlite3.connect(uri, uri=True, timeout=20)


def _restore_ship_from_backup(conn, ship_no):
    """从历史备份里把一条船整条捞回来（ships + tasks + 报工 + 排程结果）。

    不能只用最新备份：一次「恢复」操作自己也会先拍快照，如果那时 ships 行
    已经被误删，新快照里就没有它 —— 越往前的备份反而越完整，所以要逐个翻。
    """
    import glob as _g
    key = str(ship_no).strip().upper()
    cands = [p for p in _g.glob(os.path.join(DATA_DIR, "aps*.db"))
             if os.path.basename(p) != "aps.db"]
    cands.sort(key=os.path.getmtime, reverse=True)
    tables = ("ships", "tasks", "actual_progress", "realtime_progress", "plan_detail",
              "ship_group_hours", "gongxu_ship_stage", "ship_manpower")
    scanned = []
    for p in cands:
        try:
            b = _open_ro(p)
            hit = b.execute("SELECT 1 FROM ships WHERE UPPER(TRIM(ship_no))=? LIMIT 1",
                            (key,)).fetchone()
            b.close()
        except Exception as e:
            log("恢复：备份 %s 打不开（%s）" % (os.path.basename(p), e))
            continue
        scanned.append("%s%s" % (os.path.basename(p), "★" if hit else ""))
        if not hit:
            continue
        # ⚠ ATTACH 必须【先于】搬运，否则 bak.* 表还不存在，
        #   每条 INSERT 都会抛 "no such table: bak.xxx" 并被下面的 except 吞掉，
        #   结果恢复成 0 行却报告"成功"（踩过一次）
        try:
            conn.execute("DETACH DATABASE bak")     # 上一轮可能还挂着
        except Exception:
            pass
        try:
            conn.execute("ATTACH DATABASE ? AS bak", (os.path.abspath(p),))
        except Exception as e:
            log("恢复：无法挂载备份 %s（%s）" % (os.path.basename(p), e))
            continue
        total = 0
        failed = []
        for t in tables:
            try:
                cols = [r[1] for r in conn.execute("PRAGMA main.table_info(%s)" % t)]
                bcols = [r[1] for r in conn.execute("PRAGMA bak.table_info(%s)" % t)]
            except Exception:
                continue
            cols = [c for c in cols if c in bcols]
            if not cols:
                continue
            try:
                conn.execute("DELETE FROM main.%s WHERE UPPER(TRIM(ship_no))=?" % t, (key,))
                conn.execute("INSERT INTO main.%s(%s) SELECT %s FROM bak.%s "
                             "WHERE UPPER(TRIM(ship_no))=?"
                             % (t, ",".join(cols), ",".join(cols), t), (key,))
                n = conn.execute("SELECT changes() c").fetchone()[0]
                if n:
                    total += n
            except Exception as e:
                failed.append("%s(%s)" % (t, str(e)[:40]))
        conn.commit()
        try:
            conn.execute("DETACH DATABASE bak")
        except Exception:
            pass
        if total <= 0:
            return {"ok": False,
                    "detail": "在 %s 里找到这条船，但搬运数据失败：%s"
                              % (os.path.basename(p),
                                 "；".join(failed[:4]) or "列结构不兼容")}
        log("恢复船 %s：从 %s 搬回 %d 行数据%s"
            % (key, os.path.basename(p), total,
               ("（%d 张表失败：%s）" % (len(failed), "、".join(failed[:3]))) if failed else ""))
        return {"ok": True,
                "detail": "已从 %s 恢复 %d 行数据" % (os.path.basename(p), total)}
    return {"ok": False,
            "detail": "翻了 %d 个备份都没找到这条船的档案（%s）；"
                      "需在主计划执行情况表里补一行再导入"
                      % (len(scanned), "、".join(scanned) or "无备份")}


def api_suggest(h, q):
    conn = connect()
    try:
        h.json(SC.suggest_actions(conn, q.get("ship", [""])[0]))
    finally:
        conn.close()


def api_export(h, body):
    conn = connect()
    try:
        version = body.get("version") or ""
        if not version:
            v = conn.execute("SELECT version_no FROM schedule_versions WHERE is_active=1 "
                             "ORDER BY id DESC LIMIT 1").fetchone()
            version = v["version_no"] if v else ""
        if not version:
            h.json({"error": "还没有排程结果"})
            return
        path = SC.export_plan(conn, version)
        h.json({"path": path})
    finally:
        conn.close()


SCENARIO_PRESETS = [
    {"id": "base", "name": "方案A：当前人力（基准）", "def": True},
    {"id": "final2", "name": "方案B：终装+2名熟练工", "def": True},
    {"id": "sunday", "name": "方案C：周日上班+加班2H", "def": True},
    {"id": "hold", "name": "方案D：暂缓剩余工时最大的船", "def": False},
    {"id": "protect", "name": "方案E：保最高优先级船交期", "def": False},
]


def _build_scenario(conn, sid):
    if sid == "base":
        return {}, "方案A：当前人力（基准）"
    if sid == "final2":
        return {"team_delta": {"终装": {"skilled": 2}}}, "方案B：终装+2名熟练工"
    if sid == "sunday":
        return {"work_sunday": True, "overtime": 2.0}, "方案C：周日上班+加班2H"
    if sid == "hold":
        r = conn.execute("SELECT ship_no,remaining_hours FROM ships WHERE status='在制' "
                         "ORDER BY remaining_hours DESC LIMIT 1").fetchone()
        if not r:
            return {}, "方案D（无船可暂缓）"
        return {"suspend_ships": [r["ship_no"]]}, f"方案D：暂缓 {r['ship_no']}"
    if sid == "protect":
        r = conn.execute("SELECT ship_no FROM ships WHERE status='在制' ORDER BY priority "
                         "LIMIT 1").fetchone()
        if not r:
            return {}, "方案E（无船）"
        return {"ship_priority": {r["ship_no"]: 0}}, f"方案E：保 {r['ship_no']} 交期"
    return {}, sid


def api_scenario_meta(h):
    h.json({"presets": SCENARIO_PRESETS, "today": dt.date.today().isoformat()})


def api_scenario(h, body):
    conn = connect()
    try:
        ids = body.get("ids") or []
        start = to_date(body.get("start")) or dt.date.today()
        horizon = int(body.get("horizon") or 120)
        # 记住当前激活版本，跑完后恢复
        cur = conn.execute("SELECT version_no FROM schedule_versions WHERE is_active=1 "
                           "ORDER BY id DESC LIMIT 1").fetchone()
        cur_version = cur["version_no"] if cur else None
        # 方案对比属于"模拟"，跑完必须把正式结果（含船舶表预测交检）原样还原
        official_snap = SC._ship_snapshot(conn)
        official_calendar = [tuple(r) for r in conn.execute("SELECT * FROM calendar")]

        rows = []
        for sid in ids:
            scen, name = _build_scenario(conn, sid)
            scen = dict(scen)
            scen["name"] = name
            try:
                version, metrics = SC.run_schedule(conn, start=start, horizon=horizon,
                                                   freeze_days=0, scenario=scen)
                if version is None:
                    rows.append({"name": name, "score": None, "delay_ships": 0, "delay_days": 0,
                                 "unfinished": 0, "switches": 0, "peak": 0, "rows": 0})
                    continue
                ships = conn.execute("SELECT * FROM ships WHERE status='在制'").fetchall()
                key_delay = sum((s["delay_days"] or 0) for s in ships
                                if s["must_finish_on_time"] and (s["delay_days"] or 0) > 0)
                unfinished = metrics.get("unfinished", 0)
                unfinished_ships = metrics.get("unfinished_ships", 0)
                peak = max((m["peak_rate"] for m in metrics.get("team_load", [])), default=0)
                # 评分口径：排不完的船最严重（按 60 生产日折算），其次是延期船数与延期天数
                score = (unfinished_ships * 60000
                         + len(metrics.get("delay_ships", [])) * 10000
                         + metrics.get("total_delay_days", 0) * 1000
                         + key_delay * 500
                         + metrics.get("switches", 0) * 20
                         + unfinished * 10
                         + max(0, peak - 100) * 5)
                rows.append({"name": name, "score": score,
                             "delay_ships": len(metrics.get("delay_ships", [])),
                             "delay_days": metrics.get("total_delay_days", 0),
                             "unfinished": unfinished_ships, "unfinished_tasks": unfinished,
                             "switches": metrics.get("switches", 0),
                             "peak": peak, "rows": metrics.get("rows", 0)})
            except Exception as e:
                log(f"方案 {name} 运行失败: {e}")
                rows.append({"name": name, "score": None, "delay_ships": 0, "delay_days": 0,
                             "unfinished": 0, "switches": 0, "peak": 0, "rows": 0})
        rows.sort(key=lambda r: (r["score"] is None, r["score"]))
        best = next((r for r in rows if r["score"] is not None), None)
        # 恢复基准版本为激活状态，并还原船舶表预测结果（模拟不改正式结果）
        SC._restore_official(conn, cur_version, official_snap)
        conn.execute('DELETE FROM calendar')
        conn.executemany('INSERT INTO calendar VALUES(?,?,?,?,?,?)',official_calendar)
        conn.commit()
        v = conn.execute("SELECT version_no FROM schedule_versions WHERE is_active=1 "
                         "ORDER BY id DESC LIMIT 1").fetchone()
        h.json({"rows": rows, "best": best, "version": v["version_no"] if v else ""})
    finally:
        if 'official_snap' in locals():
            SC._restore_official(conn,cur_version,official_snap)
            conn.execute('DELETE FROM calendar')
            conn.executemany('INSERT INTO calendar VALUES(?,?,?,?,?,?)',official_calendar)
            conn.commit()
        conn.close()


def api_settings(h):
    conn = connect()
    try:
        teams = [dict(r) for r in conn.execute("SELECT * FROM teams ORDER BY sort_order")]
        _kg = int(float(get_setting(conn, "keel_gate", 1) or 1))
        _kr = (get_setting(conn, "keel_rule", "auto") or "auto").strip() or "auto"
        if _kr not in SC.KEEL_RULES:
            _kr = "auto"
        # 设置页需要看到所有船：在制 / 暂缓 / 冻结 / 未开工 / 完工 ——
        # 2026-10-29 PMC 要求"主计划的 100 多条船都能进来"，船底没启动的也要能管理。
        # 完工的历史船排在最后，只是可见可查，不参与排产（排产仍只取'在制'）。
        ships = _ships_rows(conn, statuses=("在制", "暂缓", "冻结", "未开工", "完工"), keel_rule=_kr)
        excl = conn.execute(
            "SELECT COUNT(*) c FROM ships WHERE status IN ('暂缓','冻结')").fetchone()["c"]
        zones = [dict(r) for r in conn.execute("SELECT * FROM zones ORDER BY zone_code")]
        hol = [dict(r) for r in conn.execute(
            "SELECT cal_date,day_type,remarks FROM calendar WHERE day_type IN ('节假日','停工') "
            "AND remarks='手工' ORDER BY cal_date")]
        h.json({"teams": teams, "ships": ships, "zones": zones,
                "excluded_count": excl,
                "keel_gate": _kg, "keel_rule": _kr,
                "worktime": get_worktime_config(conn), "holidays": hol})
    finally:
        conn.close()


def api_worktime(h, body):
    """保存作息 / 加班配置，并立即按新配置重算日历"""
    conn = connect()
    try:
        cfg = save_worktime_config(conn, body.get("cfg") or {})
        # 同步到班组表（班组级作息以 teams 为准）
        for k, col in (("work_sat", "work_sat"), ("work_sun", "work_sun"),
                       ("ot_days_per_week", "ot_days_per_week"),
                       ("ot_hours", "ot_hours"), ("new_overtime", "new_overtime")):
            if k in (body.get("cfg") or {}):
                conn.execute(f"UPDATE teams SET {col}=?", (int(float(body['cfg'][k]))
                                                           if k != "ot_hours" else
                                                           float(body["cfg"][k]),))
        conn.commit()
        SC.ensure_calendar(conn, dt.date.today(), 180,
                           cfg=get_worktime_config(conn))
        h.json({"ok": True, "cfg": cfg})
    finally:
        conn.close()


def api_holidays(h, body):
    conn = connect()
    try:
        act = body.get("act", "add")
        if act == "add":
            for d in (body.get("dates") or []):
                iso = str(d)[:10]
                conn.execute("DELETE FROM calendar WHERE cal_date=?", (iso,))
                conn.execute("INSERT INTO calendar(cal_date,day_type,is_prod,normal_hours,"
                             "overtime,remarks) VALUES(?,?,?,?,?,?)",
                             (iso, body.get("type", "节假日"), 0, 11.0, 0.0, "手工"))
        elif act == "del":
            for d in (body.get("dates") or []):
                conn.execute("DELETE FROM calendar WHERE cal_date=? AND remarks='手工'", (str(d)[:10],))
        conn.commit()
        hol = [dict(r) for r in conn.execute(
            "SELECT cal_date,day_type,remarks FROM calendar WHERE day_type IN ('节假日','停工') "
            "AND remarks='手工' ORDER BY cal_date")]
        h.json({"ok": True, "holidays": hol})
    finally:
        conn.close()


def api_whatif_add(h, body):
    """加人模拟：加 1/2/3… 人分别会怎样"""
    conn = connect()
    try:
        res = SC.simulate_add_workers(
            conn, team=body.get("team", "终装"),
            counts=tuple(body.get("counts") or (1, 2, 3, 5, 8)),
            new_workers=int(body.get("new_workers") or 0),
            ship_no=body.get("ship_no") or None,
            start=to_date(body.get("start")),
            horizon=int(body.get("horizon") or 120),
            ship_max_override=int(body.get("ship_max_override") or 0))
        h.json(res)
    finally:
        conn.close()


def api_whatif_worktime(h, body):
    """加班天数 × 周日上班 组合矩阵"""
    conn = connect()
    try:
        res = SC.simulate_worktime_matrix(
            conn, ot_days_list=tuple(body.get("ot_days_list") or (0, 2, 3, 4, 5, 6, 7)),
            sun_list=tuple(body.get("sun_list") or (0, 1)),
            ship_no=body.get("ship_no") or None,
            start=to_date(body.get("start")),
            horizon=int(body.get("horizon") or 120))
        h.json(res)
    finally:
        conn.close()


def api_manpower(h, q=None, body=None):
    """
    人力占用模式：某条船不按工序算工时，直接指定占用各班组的多少人。
    GET  ?ship=船号   -> 返回该船配置（不带 ship 则返回全部）
    POST {'act':'save', ship_no, mode, rows:[{team,workers,new_workers,start_date,end_date}]}
    """
    conn = connect()
    try:
        if body and body.get("act") == "save":
            sno = body.get("ship_no")
            if sno:
                conn.execute("UPDATE ships SET manpower_mode=? WHERE ship_no=?",
                             (int(body.get("mode") or 0), sno))
                conn.execute("DELETE FROM ship_manpower WHERE ship_no=?", (sno,))
                # segs: [{start, end, teams:{班组:{workers, new_workers}}}]
                for i, seg in enumerate(body.get("segs") or []):
                    sd = seg.get("start") or None
                    ed = seg.get("end") or None
                    for team, v in (seg.get("teams") or {}).items():
                        w = int((v or {}).get("workers") or 0)
                        if w <= 0:
                            continue
                        conn.execute(
                            "INSERT INTO ship_manpower(ship_no,team,workers,new_workers,"
                            "start_date,end_date,note,seg) VALUES(?,?,?,?,?,?,?,?)",
                            (sno, team, w, int((v or {}).get("new_workers") or 0),
                             sd, ed, seg.get("note") or "", i))
                conn.commit()
        ship = (q or {}).get("ship", [None])[0]
        if ship:
            rows = conn.execute("SELECT * FROM ship_manpower WHERE ship_no=? "
                                "ORDER BY seg, id", (ship,)).fetchall()
            s = conn.execute("SELECT ship_no,ship_type,manpower_mode,plan_inspection "
                             "FROM ships WHERE ship_no=?", (ship,)).fetchone()
            # 按 seg 归并成"时间段"
            segs, order = {}, []
            for r in rows:
                k = int(r["seg"] or 0)
                if k not in segs:
                    segs[k] = {"start": r["start_date"], "end": r["end_date"], "teams": {}}
                    order.append(k)
                segs[k]["teams"][r["team"]] = {"workers": int(r["workers"] or 0),
                                               "new_workers": int(r["new_workers"] or 0)}
            h.json({"rows": [dict(r) for r in rows],
                    "segs": [segs[k] for k in order],
                    "ship": dict(s) if s else {}})
        else:
            rows = conn.execute("SELECT * FROM ship_manpower ORDER BY ship_no, team").fetchall()
            ships = conn.execute("SELECT ship_no,ship_type,manpower_mode FROM ships "
                                 "WHERE status='在制' ORDER BY ship_no").fetchall()
            h.json({"rows": [dict(r) for r in rows], "ships": [dict(s) for s in ships]})
    finally:
        conn.close()


def api_gongxu(h, q=None, body=None):
    """工序工位文件：查询五组总工时 + 待确认船号；支持手动把船号加入总计划。

    GET  /api/gongxu                 列出全部五组工时与待确认船号
    POST /api/gongxu {act:'add',ship_nos:[...]}     把勾选的船号加入总计划
    POST /api/gongxu {act:'del',ship_no:...}        删除一条工序工位记录
    """
    conn = connect()
    try:
        init_db(conn)
        body = body or {}
        act = body.get("act") or "list"

        if act == "add":
            now = dt.date.today()
            added, existed = [], []
            _blk = _excluded_set(conn)
            for sn in (body.get("ship_nos") or []):
                sn = str(sn).strip().upper()
                if not sn:
                    continue
                if sn in _blk:
                    existed.append(sn + "(已作废)")
                    continue
                old = conn.execute("SELECT ship_no FROM ships WHERE ship_no=?", (sn,)).fetchone()
                st = conn.execute("SELECT ship_type,total_hours FROM gongxu_ship_stage "
                                  "WHERE ship_no=? ORDER BY imported_at DESC LIMIT 1",
                                  (sn,)).fetchone()
                ship_type = (st["ship_type"] if st else "") or ""
                if old:
                    existed.append(sn)
                    conn.execute("UPDATE ships SET ship_type=COALESCE(NULLIF(?, ''), ship_type) "
                                 "WHERE ship_no=?", (ship_type, sn))
                else:
                    # 其余时间节点留空，由用户在【总计划】页补填。
                    # 状态必须用系统认可的枚举值（在制/完工/暂缓/冻结）—— 没有"未开工"这个状态，
                    # 用错值会让船在列表里查不到，用户以为"加了但没生效"。
                    # 用"在制"：船会立刻出现在总计划里可编辑；因为没填船底启动日期，
                    # 排产时会被船底启动门禁拦住，不会误排进周计划。
                    conn.execute(
                        "INSERT INTO ships(ship_no,ship_type,status,priority,manual_priority,"
                        "must_finish_on_time,allow_delay,remarks) "
                        "VALUES(?,?,?,2,'正常',0,1,?)",
                        (sn, ship_type, "在制", "来源：工序工位文件导入（时间节点待补填）"))
                    added.append(sn)
                conn.execute("UPDATE gongxu_ship_stage SET added=1 WHERE ship_no=?", (sn,))
            conn.commit()
            h.json({"ok": True, "added": added, "existed": existed})
            return

        if act == "del":
            sn = str(body.get("ship_no") or "").strip().upper()
            conn.execute("DELETE FROM ship_group_hours WHERE ship_no=?", (sn,))
            conn.execute("DELETE FROM gongxu_ship_stage WHERE ship_no=?", (sn,))
            conn.commit()
            h.json({"ok": True, "deleted": sn})
            return

        # ---- list ----
        raw = conn.execute("SELECT * FROM ship_group_hours ORDER BY ship_no, group_code").fetchall()
        stage = conn.execute("SELECT * FROM gongxu_ship_stage "
                             "ORDER BY imported_at DESC, ship_no").fetchall()
        orders = {g[0]: i for i, g in enumerate(GX_GROUPS)}

        by_ship = {}
        for r in raw:
            d = dict(r)
            by_ship.setdefault(r["ship_no"], {"ship_no": r["ship_no"],
                                              "ship_type": r["ship_type"],
                                              "src_file": r["src_file"], "rev": r["rev"],
                                              "groups": {}, "total": 0.0})["groups"][
                r["group_code"]] = round(float(r["hours"] or 0), 2)
        for sn, d in by_ship.items():
            d["total"] = round(sum(d["groups"].values()), 2)
            d["group_list"] = [{"code": g[0], "hours": d["groups"].get(g[0], 0.0)}
                               for g in GX_GROUPS]

        rows = sorted(by_ship.values(), key=lambda x: (x["ship_type"] or "", x["ship_no"]))

        stages = []
        for r in stage:
            try:
                gj = json.loads(r["group_json"] or "{}")
            except Exception:
                gj = {}
            gr = gj.get("groups") or {}
            stages.append({"ship_no": r["ship_no"], "ship_type": r["ship_type"],
                           "src_file": r["src_file"], "rev": r["rev"],
                           "total_hours": round(float(r["total_hours"] or 0), 2),
                           "groups": {g[0]: round(float(gr.get(g[0]) or 0), 2) for g in GX_GROUPS},
                           "added": int(r["added"] or 0),
                           "imported_at": r["imported_at"],
                           "exists": bool(conn.execute("SELECT 1 FROM ships WHERE ship_no=?",
                                                       (r["ship_no"],)).fetchone())})
        h.json({"groups": [g[0] for g in GX_GROUPS], "rows": rows, "stages": stages,
                "count": len(rows)})
    finally:
        conn.close()


def api_upload(h, body):
    """手动上传 Excel（base64），保存并按内容自动识别导入"""
    conn = connect()
    try:
        name = (body or {}).get("name") or ""
        data = (body or {}).get("data") or ""
        if not name or not data:
            h.json({"error": "缺少文件名或文件内容"})
            return
        res = IM.import_uploaded_file(conn, name, data, auto_import=True)
        h.json(res)
    finally:
        conn.close()


def api_realtime_files(h, q=None, body=None):
    """列出当前自动识别到的单船完成情况表；也可手动指定扫描目录"""
    conn = connect()
    try:
        if body and body.get("act") == "setdir":
            conn.execute("INSERT OR REPLACE INTO settings(key,value) VALUES('realtime_dir',?)",
                         (body.get("path") or "",))
            conn.commit()
        files = IM.scan_realtime_files(conn)
        custom = conn.execute("SELECT value FROM settings WHERE key='realtime_dir'").fetchone()
        dirs = IM.find_realtime_dirs()
        # 只按文件名列出来（解析 Excel 很慢，留到真正导入时再做）
        h.json({"files": [{"name": os.path.basename(p), "path": p,
                           "kind": "realtime"} for p in files[:200]],
                "total": len(files),
                "custom_dir": custom["value"] if custom else "",
                "scan_dirs": dirs[:8]})
    finally:
        conn.close()


def api_import_realtime(h, body):
    """只导入实时/完成情况表并做交叉核对（不重跑全量导入）"""
    conn = connect()
    try:
        files = [(body or {}).get("path")] if (body or {}).get("path") else IM.scan_realtime_files(conn)
        outs = []
        for p in files:
            try:
                outs.append(IM.import_realtime_progress(conn, p))
            except Exception as e:
                outs.append({"file": os.path.basename(p), "error": str(e)})
        cross = IM.merge_cross_progress(conn)
        h.json({"imported": outs, "cross": cross})
    finally:
        conn.close()


def api_estimate(h, q=None, body=None):
    """
    交检预测（纯工时法，不做工序排程）。
    GET  ?teams=终装&teams=甲板   -> 不传则五个班组全部
    GET  ?mode=queue|ideal|jit|normal_jit
    POST {'act':'export','teams':[...]} -> 导出 Excel
    """
    conn = connect()
    try:
        mode = ((q or {}).get("mode", [None])[0]) or (body or {}).get("mode") or "queue"
        if mode not in ("queue", "ideal", "jit", "normal_jit"):
            mode = "queue"
        if body and body.get("act") == "export":
            path = SC.export_estimate(conn, teams=body.get("teams") or None, mode=mode)
            h.json({"path": path})
            return
        teams = (q or {}).get("teams") or (body or {}).get("teams") or None
        h.json(SC.estimate_finish_by_hours(conn, teams, mode=mode))
    finally:
        conn.close()


def api_audit(h, q=None, body=None):
    """
    工时核算对账明细。
    GET  ?ship=A70L007&team=初装 -> 返回明细行（供对账）
    POST {'act':'export','ship':..,'team':..} -> 导出 Excel
    """
    conn = connect()
    try:
        if body and body.get("act") == "export":
            path = SC.export_audit(conn, ship_no=(body.get("ship") or None),
                                   team=(body.get("team") or None))
            h.json({"path": path})
            return
        rows = SC.audit_rows(conn, (q or {}).get("ship", [None])[0],
                             (q or {}).get("team", [None])[0])
        h.json({"rows": rows[:3000], "total": len(rows)})
    finally:
        conn.close()


def api_hour_rules(h, q=None, body=None):
    """工时统计规则的开关：班组情况表未列的工序是否不统计"""
    conn = connect()
    try:
        if body and body.get("act") == "set":
            set_setting(conn, "team_scope_only", int(body.get("team_scope_only", 1)))
            IM.restore_expired_ships(conn) if False else None
            # 立即重算：先把两类归零的恢复成定额，再按规则重新归零
            conn.execute(
                "UPDATE tasks SET remaining_hours=planned_hours "
                "WHERE status<>'已完成' AND remaining_hours=0 AND planned_hours>0")
            conn.commit()
            IM.apply_team_scope(conn)
            IM.apply_option_flag(conn)
        h.json({"team_scope_only": int(float(get_setting(conn, "team_scope_only", 1) or 1))})
    finally:
        conn.close()


def api_temp_tasks(h, q=None, body=None):
    """
    临时任务（某班组在某段时间临时增加的工时）。
    GET  -> 列表
    POST {'act':'add'|'update'|'del'|'toggle', ...}
    """
    conn = connect()
    try:
        init_db(conn)
        if body and body.get("act") == "add":
            conn.execute("INSERT INTO temp_tasks(team,start_date,end_date,hours,note,enabled)"
                         " VALUES(?,?,?,?,?,1)",
                         (body.get("team") or "", body.get("start") or None,
                          body.get("end") or None, float(body.get("hours") or 0),
                          body.get("note") or ""))
            conn.commit()
        elif body and body.get("act") == "update":
            conn.execute("UPDATE temp_tasks SET team=?,start_date=?,end_date=?,hours=?,note=? "
                         "WHERE id=?",
                         (body.get("team") or "", body.get("start") or None,
                          body.get("end") or None, float(body.get("hours") or 0),
                          body.get("note") or "", int(body.get("id") or 0)))
            conn.commit()
        elif body and body.get("act") == "del":
            conn.execute("DELETE FROM temp_tasks WHERE id=?", (int(body.get("id") or 0),))
            conn.commit()
        elif body and body.get("act") == "toggle":
            conn.execute("UPDATE temp_tasks SET enabled=? WHERE id=?",
                         (int(body.get("enabled", 1)), int(body.get("id") or 0)))
            conn.commit()
        rows = [dict(r) for r in conn.execute(
            "SELECT * FROM temp_tasks ORDER BY COALESCE(start_date,'9999'), id")]
        h.json({"rows": rows})
    finally:
        conn.close()


def api_headcount(h, q=None, body=None):
    """
    五组综合人力建议。
    GET  ?tolerance=0  -> 各班组需要加多少人才能满足主计划
    POST {'act':'sim','plan':{'初装':2,...},'horizon':120} -> 按方案跑排程看还剩几艘延期
    """
    conn = connect()
    try:
        if body and body.get("act") == "sim":
            h.json(SC.simulate_headcount_plan(
                conn, plan=body.get("plan") or {}, start=to_date(body.get("start")),
                horizon=_num_param(body, "horizon", 120, lo=1, hi=730),
                freeze_days=_num_param(body, "freeze", 0, lo=0, hi=90)))
            return
        tol = int((q or {}).get("tolerance", [0])[0] or (body or {}).get("tolerance") or 0)
        h.json(SC.suggest_headcount(conn, tolerance_days=tol))
    finally:
        conn.close()


def api_expired_ships(h, q=None, body=None):
    """
    过期船（计划交检早于今天）的开关与管理。
    GET  -> 返回当前开关、被跳过的船列表
    POST -> {'act':'set','skip':0/1} 改开关并立即重算
            {'act':'restore','ships':[...]} 恢复为在制（不传则全部恢复）
    """
    conn = connect()
    try:
        if body:
            act = body.get("act")
            if act == "set":
                set_setting(conn, "skip_expired_ships", int(body.get("skip", 1)))
                IM.restore_expired_ships(conn)          # 先全部恢复
                if int(body.get("skip", 1)):
                    IM.skip_expired_ships(conn)         # 再按开关重新判定
            elif act == "restore":
                IM.restore_expired_ships(conn, body.get("ships") or None)
        skip = int(float(get_setting(conn, "skip_expired_ships", 1) or 1))
        h.json({"skip_expired": skip, "expired": IM.expired_ships(conn)})
    finally:
        conn.close()


def api_keel_gate(h, q=None, body=None, conn=None):
    """
    船底启动门禁 —— 船底未启动（日期未到 / 没填）的船不参与排产，也就不进周计划。

    GET  -> 当前开关、判定口径、以及按当前口径会被排除的船清单
    POST -> {'act':'set','enabled':0/1,'rule':'auto|plan|sale'} 修改设置

    口径说明（rule）：
      auto（默认）：实际船底启动日有值 -> 已启动；否则要求计划启动日 <= 排产基准日
      plan        ：只认计划船底启动日
      sale        ：只认实际船底启动日
    """
    own = conn is None
    if own:
        conn = connect()
    try:
        if body:
            act = body.get("act") or "set"
            if act == "set":
                if body.get("enabled") is not None:
                    set_setting(conn, "keel_gate",
                                1 if int(body.get("enabled") or 0) else 0)
                rule = (body.get("rule") or "").strip()
                if rule in SC.KEEL_RULES:
                    set_setting(conn, "keel_rule", rule)
                log("船底启动门禁设置已更新")
        gate = int(float(get_setting(conn, "keel_gate", 1) or 1))
        rule = (get_setting(conn, "keel_rule", "auto") or "auto").strip() or "auto"
        if rule not in SC.KEEL_RULES:
            rule = "auto"
        base = dt.date.today()
        blocked = []
        for r in conn.execute("SELECT * FROM ships WHERE status='在制'"):
            d = dict(r)
            ok, why, sd = SC.keel_status(d, base, rule)
            if not ok:
                blocked.append({"ship_no": d["ship_no"], "ship_type": d.get("ship_type") or "",
                                "plan_start": d.get("plan_start"),
                                "sale_start": d.get("sale_start"),
                                "plan_inspection": d.get("plan_inspection"),
                                "remaining_hours": float(d.get("remaining_hours") or 0),
                                "reason": why})
        h.json({"keel_gate": gate, "keel_rule": rule, "rules": list(SC.KEEL_RULES),
                "base_day": base.isoformat(),
                "blocked": blocked, "blocked_count": len(blocked)})
    finally:
        if own:
            conn.close()


# 排产口径 -> 中文标签
# 标准 = 舱室上人限制全开（单船并行人数 + 同舱区同时作业班组数）
# 理想 = 放开【单船并行人数上限 + 同舱区班组数上限】，但【单工序人数上限 4 人仍保留】
#        （2026-10-15 用户确认）
SCHEDULE_MODE_LABEL = {
    "normal": "标准·含舱室上人限制",
    "ideal": "理想·放开单船人数（单工序仍限4人）",
    "jit": "理想+JIT",
    "normal_jit": "标准+JIT",
}
SCHEDULE_MODE_TAG = {"normal": "标准", "ideal": "理想", "jit": "理想JIT", "normal_jit": "标准JIT"}


def version_modes(conn, limit=30):
    """最近 N 个版本的口径标签，供前端下拉切换（标准 / 理想 两套计划并存对照）。

    老版本 params 里没有 mode 字段，按 algorithm 兜底为"标准"。
    """
    out = []
    for r in conn.execute(
            "SELECT version_no,created_at,algorithm,params,is_active "
            "FROM schedule_versions ORDER BY id DESC LIMIT ?", (limit,)):
        try:
            mode = (json.loads(r["params"] or "{}") or {}).get("mode") or ""
        except (ValueError, TypeError):
            mode = ""
        if mode not in SCHEDULE_MODE_LABEL:
            mode = "normal"          # 旧版本无口径字段，视为标准
        out.append({
            "version_no": r["version_no"],
            "created_at": r["created_at"],
            "algorithm": r["algorithm"],
            "mode": mode,
            "mode_label": SCHEDULE_MODE_LABEL[mode],
            "mode_tag": SCHEDULE_MODE_TAG[mode],
            "is_active": int(r["is_active"] or 0),
        })
    return out


def _resolve_version(conn, want):
    """按前端传的 version_no 取版本；没传或不存在则退回当前正式版本。"""
    want = (want or "").strip()
    if want:
        v = conn.execute("SELECT version_no FROM schedule_versions WHERE version_no=?",
                         (want,)).fetchone()
        if v:
            return v
    return conn.execute("SELECT version_no FROM schedule_versions WHERE is_active=1 "
                        "ORDER BY id DESC LIMIT 1").fetchone()


def api_teamplan(h, q):
    """某个班组的未来 7 天周计划 / 下月月计划

    version 参数可选：指定按哪个排产版本出计划（标准 / 理想 两套并存对照）；
    不传则用当前正式版本。
    """
    conn = connect()
    try:
        team = (q.get("team") or ["初装"])[0]
        mode = (q.get("mode") or ["week"])[0]
        base = to_date((q.get("date") or [None])[0])
        s0 = to_date((q.get("start") or [None])[0])
        s1 = to_date((q.get("end") or [None])[0])
        v = _resolve_version(conn, (q.get("version") or [""])[0])
        if not v:
            h.json({"error": "尚未排产，请先到【排产与甘特图】排产"})
            return
        out = SC.team_plan(conn, v["version_no"], team, mode, base, s0, s1)
        out["version"] = v["version_no"]
        out["versions"] = version_modes(conn)
        h.json(out)
    finally:
        conn.close()


def api_export_teamplan(h, body):
    """导出五个班组的周计划 / 月计划 Excel

    body.version 可选：导出指定排产版本的计划；不传则用当前正式版本。
    文件名带口径标签，避免标准版与理想版互相覆盖。
    """
    conn = connect()
    try:
        mode = (body or {}).get("mode") or "week"
        v = _resolve_version(conn, (body or {}).get("version"))
        if not v:
            h.json({"error": "尚未排产"})
            return
        version = v["version_no"]
        tag = next((x["mode_tag"] for x in version_modes(conn)
                    if x["version_no"] == version), "")
        label = "周" if mode == "week" else "月"
        path = os.path.join(OUT_DIR, f"班组{label}计划_{tag}_{version}.xlsx")
        path = SC.export_team_plan(conn, version, mode, path=path)
        h.json({"path": path})
    finally:
        conn.close()


def api_gap(h):
    """产能缺口分析（按周 × 班组）"""
    conn = connect()
    try:
        v = conn.execute("SELECT version_no,metrics FROM schedule_versions WHERE is_active=1 "
                         "ORDER BY id DESC LIMIT 1").fetchone()
        gap = []
        if v and v["metrics"]:
            try:
                gap = json.loads(v["metrics"]).get("gap_by_week") or []
            except ValueError:
                gap = []
        h.json({"version": v["version_no"] if v else "", "gap": gap})
    finally:
        conn.close()


def api_save_teams(h, body):
    conn = connect()
    try:
        # 前端字段名 -> 数据库列名
        key_map = {"skilled": "skilled", "new_workers": "new_workers",
                   "sh": "sh_per_day", "nh": "nh_per_day",
                   "work_sat": "work_sat", "work_sun": "work_sun",
                   "overtime": "overtime", "ship_max": "ship_max", "ship_max_u": "ship_max_u"}
        for code, f in (body.get("teams") or {}).items():
            sets, vals = [], []
            for k, v in f.items():
                col = key_map.get(k)
                if not col:
                    continue
                sets.append(f"{col}=?")
                vals.append(to_number(v, 0) if not isinstance(v, bool) else (1 if v else 0))
            if sets:
                conn.execute(f"UPDATE teams SET {','.join(sets)} WHERE team_code=?", vals + [code])
        conn.commit()
        h.json({"ok": True})
    finally:
        conn.close()


def api_save_ships(h, body):
    """
    保存船只的优先级 / 状态 / 固定作业人数 / 时间节点。

    每个船只传一个字段字典 f，支持的键：
      p   = 人工优先级（紧急/高/正常/低/暂缓/冻结）
      s   = 状态（在制/完工/暂缓/冻结）
      m   = 必须按期（0/1）
      fw  = 固定作业人数（总）
      fn  = 其中新人
      pi  = 手填交检日（写 manual_inspection + plan_inspection）
      ps  = 计划船底启动日（写 plan_start，空串=清空）   <- 2026-10-10 新增
      ss  = 实际船底启动日（写 sale_start，空串=清空）   <- 2026-10-10 新增
    """
    conn = connect()
    try:
        for sno, f in (body.get("ships") or {}).items():
            f = f or {}
            prio = PRIORITY_MAP.get(f.get("p", "正常"), 2)
            fw = int(f.get("fw") or 0)
            fn = int(f.get("fn") or 0)
            if fw > 0:
                fn = min(fn, fw)
            else:
                fn = 0
            # pi = 手填交检日；保存后同时同步到 plan_inspection，并在以后导入时保持优先
            pi = (f.get("pi") or "").strip() or None
            sets = ["manual_priority=?", "priority=?", "must_finish_on_time=?", "status=?",
                    "fixed_workers=?", "fixed_new=?", "manual_inspection=?",
                    "plan_inspection=COALESCE(?,plan_inspection)"]
            args = [f.get("p", "正常"), prio, int(bool(f.get("m"))), f.get("s", "在制"),
                    fw, fn, pi, pi]
            # 船底启动日：只有显式传了键才动（传空串 = 清空那一天）
            for key, col in (("ps", "plan_start"), ("ss", "sale_start")):
                if key in f:
                    v = (f.get(key) or "").strip() or None
                    if v:
                        try:
                            dt.date.fromisoformat(v[:10])
                            v = v[:10]
                        except ValueError:
                            h.json({"error": "%s 日期格式不对：%r" % (col, f.get(key))})
                            return
                    sets.append("%s=?" % col)
                    args.append(v)
            args.append(sno)
            conn.execute("UPDATE ships SET %s WHERE ship_no=?" % ",".join(sets), args)
        conn.commit()
        h.json({"ok": True})
    finally:
        conn.close()


def api_ships_bulk(h, body, conn=None):
    """
    批量剔除 / 恢复船只 —— 几十条船一键筛选，不必逐行点状态下拉。

    POST body:
      {'act':'exclude','ships':[...]}    把列出的船剔除（不参与排产）
      {'act':'include','ships':[...]}    把列出的船放回（恢复参与排产）
      {'act':'keep_only','ships':[...]}  只保留列出的船参与排产，其余全部剔除
      {'act':'exclude_all'}              所有在制船全部剔除
      {'act':'include_all'}              所有被剔除的船全部放回

    ships 可以是数组，也可以是一整段文本（逗号 / 分号 / 空格 / 换行分隔），
    船号大小写不敏感。返回命中数、未找到的船号，以及最新计数。

    实现口径：用 '暂缓' 表示"剔除、不参与排产"。这是系统既有语义，
    排产只取 status='在制' 的船，且导入数据时 '暂缓'/'冻结' 不会被自动改回，
    因此剔除状态可以稳定保持，随时可逆。
    """
    own = conn is None
    if own:
        conn = connect()
    try:
        act = (body.get("act") or "").strip()
        raw = body.get("ships") or []
        if isinstance(raw, str):
            raw = re.split(r"[,，;；、\s]+", raw)
        want = [str(x).strip() for x in raw if str(x).strip()]

        all_rows = conn.execute("SELECT ship_no, status FROM ships").fetchall()
        real = {}
        for r in all_rows:
            real[str(r["ship_no"]).strip().upper()] = (r["ship_no"], r["status"])
        hit, miss = [], []
        for w in want:
            k = w.upper()
            if k in real:
                hit.append(real[k][0])
            else:
                miss.append(w)

        changed = 0
        if act == "exclude":
            for sno in hit:
                changed += conn.execute(
                    "UPDATE ships SET status='暂缓' WHERE ship_no=? AND status<>'暂缓'",
                    (sno,)).rowcount or 0
        elif act == "include":
            for sno in hit:
                changed += conn.execute(
                    "UPDATE ships SET status='在制' "
                    "WHERE ship_no=? AND status IN ('暂缓','冻结')", (sno,)).rowcount or 0
        elif act == "keep_only":
            keep = {s.upper() for s in hit}
            for r in all_rows:
                sno, st = r["ship_no"], (r["status"] or "")
                if st not in ("在制", "暂缓", "冻结"):
                    continue                      # '完工' 等业务状态不动
                if str(sno).strip().upper() in keep:
                    changed += conn.execute(
                        "UPDATE ships SET status='在制' WHERE ship_no=? AND status<>'在制'",
                        (sno,)).rowcount or 0
                else:
                    changed += conn.execute(
                        "UPDATE ships SET status='暂缓' WHERE ship_no=? AND status<>'暂缓'",
                        (sno,)).rowcount or 0
        elif act == "exclude_all":
            changed = conn.execute(
                "UPDATE ships SET status='暂缓' WHERE status='在制'").rowcount or 0
        elif act == "include_all":
            changed = conn.execute(
                "UPDATE ships SET status='在制' "
                "WHERE status IN ('暂缓','冻结')").rowcount or 0
        else:
            h.json({"error": "未知操作: %s" % act})
            return

        conn.commit()
        in_plan = conn.execute(
            "SELECT COUNT(*) c FROM ships WHERE status='在制'").fetchone()["c"]
        excluded = conn.execute(
            "SELECT COUNT(*) c FROM ships WHERE status IN ('暂缓','冻结')").fetchone()["c"]
        log(f"批量船只操作 {act}：变动 {changed} 艘，参与排产 {in_plan} 艘，已剔除 {excluded} 艘")
        h.json({"ok": True, "act": act, "changed": changed,
                "in_plan": in_plan, "excluded": excluded,
                "matched": len(hit), "not_found": miss})
    finally:
        if own:
            conn.close()


# ============================================================ 删除船只
# 误操作加进来的船要能一键清掉；删干净（连工序一起）还是只删船，由调用方决定。
_SHIP_CHILD_TABLES = [
    # (表名, 船号列名)  —— 删船时要一起清掉的"从属数据"
    ("tasks", "ship_no"),
    ("actual_progress", "ship_no"),
    ("realtime_progress", "ship_no"),
    ("schedule_result", "ship_no"),
    ("ship_manpower", "ship_no"),
    ("ship_group_hours", "ship_no"),
    ("gongxu_ship_stage", "ship_no"),
]


def _ship_exists(conn, ship_no):
    """按大小写+去空格匹配，返回库内真实的船号写法；没有返回 None"""
    if ship_no is None:
        return None
    key = str(ship_no).strip().upper()
    if not key:
        return None
    for r in conn.execute("SELECT ship_no FROM ships"):
        if str(r["ship_no"]).strip().upper() == key:
            return r["ship_no"]
    return None


# ---------------- 作废船黑名单 ----------------
def _excluded_set(conn):
    """已作废（界面上叉掉）的船号集合 —— 忽略大小写和空格"""
    try:
        return {str(r[0]).strip().upper() for r in
                conn.execute("SELECT ship_no FROM excluded_ships")}
    except Exception:
        return set()


def is_excluded(conn, ship_no):
    if not ship_no:
        return False
    return str(ship_no).strip().upper() in _excluded_set(conn)


def exclude_ship(conn, ship_no, reason="界面作废"):
    """把船号写进黑名单（并从 ships 表移除）；已在黑名单则忽略"""
    if not ship_no:
        return False
    key = str(ship_no).strip().upper()
    conn.execute("INSERT OR REPLACE INTO excluded_ships(ship_no,reason,created_at) "
                 "VALUES(?,?,?)", (key, reason, dt.datetime.now().isoformat(timespec="seconds")))
    log(f"作废船：{key}（{reason}）—— 以后导入不会再把它复活")
    return True


def restore_ship(conn, ship_no):
    """从黑名单移除（下次导入时如果源文件还有它，就会重新建回来）"""
    key = str(ship_no).strip().upper()
    n = conn.execute("DELETE FROM excluded_ships WHERE UPPER(TRIM(ship_no))=?", (key,)).rowcount
    conn.commit()
    return n


def list_excluded(conn):
    return [dict(r) for r in conn.execute(
        "SELECT ship_no,reason,created_at FROM excluded_ships ORDER BY created_at DESC")]


def delete_ship(conn, ship_no, with_tasks=True, dry_run=False, block=True):
    """
    删除一条船。默认连它的工序/报工/排程结果一起删干净。

    :param block: True（默认）同时写入作废黑名单，导入时不会再复活这条船。

    :param ship_no:   船号（大小写、首尾空格不敏感）
    :param with_tasks: True 连 from 从属数据一起删；False 只删 ships 那一行
    :param dry_run:    True 只统计会删掉什么，不真删（给前端弹窗预览用）

    :return: dict；失败返回 {"error": ...}
    """
    real = _ship_exists(conn, ship_no)
    if real is None:
        return {"error": "船不存在：%s" % ship_no}

    row = conn.execute("SELECT * FROM ships WHERE ship_no=?", (real,)).fetchone()
    info = {"ship_no": real, "ship_type": row["ship_type"] if row else "",
            "status": row["status"] if row else ""}

    # 统计会被一起删掉的从属数据
    counts = {}
    for tbl, col in _SHIP_CHILD_TABLES:
        try:
            n = conn.execute("SELECT COUNT(*) c FROM %s WHERE %s=?" % (tbl, col),
                             (real,)).fetchone()["c"]
        except Exception:
            n = 0
        if n:
            counts[tbl] = int(n)
    info["child_counts"] = counts
    info["tasks"] = counts.get("tasks", 0)
    hours = 0.0
    try:
        hours = float(conn.execute(
            "SELECT COALESCE(SUM(planned_hours),0) h FROM tasks WHERE ship_no=?",
            (real,)).fetchone()["h"] or 0)
    except Exception:
        pass
    info["hours"] = round(hours, 1)

    if dry_run:
        info["ok"] = True
        info["dry_run"] = True
        return info

    # 真正删除。整个删船动作放一个事务里，避免删一半失败留下孤儿工序。
    # 注意：调用方可能已经开了事务（导入流程里就是这样），此时再 BEGIN 会报错，
    # 所以这里不用显式 BEGIN —— sqlite3 的隐式事务已经够用，出错时 rollback。
    try:
        if with_tasks:
            for tbl, col in _SHIP_CHILD_TABLES:
                try:
                    conn.execute("DELETE FROM %s WHERE %s=?" % (tbl, col), (real,))
                except Exception as e:
                    log(f"删船：清理 {tbl} 失败（跳过）：{e}")
        conn.execute("DELETE FROM ships WHERE ship_no=?", (real,))
        conn.commit()
        if block:
            exclude_ship(conn, real, reason="界面删除")
            conn.commit()
    except Exception as e:
        try:
            conn.rollback()
        except Exception:
            pass
        return {"error": "删除失败：%s" % e}

    info["ok"] = True
    info["deleted"] = real
    info["with_tasks"] = bool(with_tasks)
    info["blocked"] = bool(block)
    log(f"删除船：{real}（船型 {info['ship_type']}，状态 {info['status']}，"
        f"{'连工序' if with_tasks else '只删船'}，清理 {info['child_counts']}）")
    return info


def api_ship_delete(h, body, conn=None):
    """
    删除船（误操作加进来的船一键清掉）。

    POST body:
      {'act':'preview','ships':['A46C 006']}                  只预览会删掉什么
      {'act':'delete','ships':[...],'with_tasks':1}           执行删除（默认连工序）
      {'act':'delete','ship_no':'A46C 006','with_tasks':0}    只删这一条、保留工序
      {'act':'fix_dup','ships':[...]}                         把带空格的脏船号规整掉（合并同号）

    ships 可以是数组，也可以是一整段用逗号/分号/换行分隔的文本。
    船号大小写、首尾空格不敏感。
    """
    own = conn is None
    if own:
        conn = connect()
    try:
        body = body or {}
        act = (body.get("act") or "delete").strip()

        raw = body.get("ships")
        if raw is None and body.get("ship_no") is not None:
            raw = [body.get("ship_no")]
        if isinstance(raw, str):
            raw = re.split(r"[,，;；、\n\r]+", raw)
        want = [str(x).strip() for x in (raw or []) if str(x).strip()]
        if not want:
            h.json({"error": "没有指定要删除的船号"})
            return

        if act == "preview":
            out = []
            for w in want:
                out.append(delete_ship(conn, w, dry_run=True))
            h.json({"ok": True, "act": "preview", "items": out})
            return

        if act == "fix_dup":
            # 把船号里的空格（含全角空格 / 制表符）去掉后与已有船合并
            fixed, failed = [], []
            for w in want:
                real = _ship_exists(conn, w)
                if real is None:
                    failed.append(w)
                    continue
                clean = re.sub(r"\s+", "", str(real)).upper()
                if clean == real:
                    continue                       # 本来就是干净的
                if _ship_exists(conn, clean):
                    failed.append("%s（清理后 %s 已存在，需手动处理）" % (real, clean))
                    continue
                conn.execute("UPDATE ships SET ship_no=? WHERE ship_no=?", (clean, real))
                for tbl, col in _SHIP_CHILD_TABLES:
                    try:
                        conn.execute("UPDATE %s SET %s=? WHERE %s=?" % (tbl, col, col),
                                     (clean, real))
                    except Exception:
                        pass
                fixed.append("%s -> %s" % (real, clean))
            conn.commit()
            h.json({"ok": True, "act": "fix_dup", "fixed": fixed, "failed": failed})
            return

        # 执行删除
        with_tasks = bool(body.get("with_tasks", 1))
        results = [delete_ship(conn, w, with_tasks=with_tasks) for w in want]
        ok = [r for r in results if r.get("ok")]
        bad = [r for r in results if not r.get("ok")]
        in_plan = conn.execute(
            "SELECT COUNT(*) c FROM ships WHERE status='在制'").fetchone()["c"]
        h.json({"ok": True, "act": "delete", "deleted": len(ok),
                "items": results, "errors": [r.get("error") for r in bad],
                "in_plan": in_plan})
    finally:
        if own:
            conn.close()


# 复制船时"沿用源船"的时间节点字段（不含 predicted_finish / 各类统计量）
_SHIP_DATE_FIELDS = ["plan_start", "plan_demould", "plan_deck_bond", "plan_hardtop",
                     "plan_inspection", "sale_start", "sale_demould"]


def _ship_type_prefix(ship_type):
    """从船型名里剥掉 REV 等后缀，得到船号用的前缀：'A36MC Rev1' -> 'A36MC'"""
    s = str(ship_type or "").strip()
    s = re.split(r"\s*REV", s, flags=re.I)[0]
    return s.strip().upper()


def next_ship_no(conn, ship_type, exclude=None):
    """
    按船型扫全库，给出下一个可用船号（最大序号 +1，序号补足 3 位）。

    船号规律：船型前缀 + 3 位序号（A45S003 ~ A45S008 -> A45S009）。
    只在"用户没手填序号"时作为建议值；手填的序号始终优先。
    """
    pfx = _ship_type_prefix(ship_type)
    if not pfx:
        return ""
    top = 0
    width = 3
    for r in conn.execute("SELECT ship_no FROM ships"):
        sn = str(r["ship_no"] or "").strip().upper()
        m = re.match(r"^([A-Z]+[0-9]*[A-Z]*?)(\d{2,})$", sn)
        if not m:
            continue
        # 同前缀（含族级前缀，A36MC 与 A36MCC 不互相干扰）
        if m.group(1) != pfx:
            continue
        num = int(m.group(2))
        width = max(width, len(m.group(2)))
        if num > top:
            top = num
    if top == 0:
        return ""
    cand = "%s%0*d" % (pfx, width, top + 1)
    used = {str(x["ship_no"]).strip().upper() for x in
            conn.execute("SELECT ship_no FROM ships")}
    while cand in used:
        top += 1
        cand = "%s%0*d" % (pfx, width, top + 1)
    return cand


def copy_ship(conn, src_ship_no, new_ship_no, new_ship_type="", task_mode="template",
              copy_dates=False):
    """
    复制一艘船：同型号换个序号就是一条新船，工时按新船船型重新刷新。

    :param src_ship_no:  源船号（必须已存在）
    :param new_ship_no:  新船号（不能与库内任何船重号）
    :param new_ship_type: 新船船型；留空则沿用源船船型
    :param task_mode:    "template" = 按新船船型从工艺模板重新展开全部工序（工时全新、全部未开工）
                         "clone"    = 照搬源船的工序清单，但完成量清零（模板数据不全时用）
    :param copy_dates:   True 沿用源船时间节点；False（默认）全部留空由用户补填

    :return: dict 结果；出错返回 {"error": ...}
    """
    src = conn.execute("SELECT * FROM ships WHERE ship_no=?", (src_ship_no,)).fetchone()
    if src is None:
        return {"error": "源船不存在：%s" % src_ship_no}
    # 船号规范化：去掉全部空格（含全角空格/制表符）+ 转大写。
    # 不规范化的话「A46C 006」会被当成另一条船存进去，之后剔除了还会再冒出来。
    new_no = norm_ship_no(new_ship_no)
    if not new_no:
        return {"error": "新船号不能为空"}
    if re.search(r"\s", str(new_ship_no or "")):
        # 输入里带空格：告诉用户已经帮你清掉了，避免他以为输错了没生效
        log(f"复制船：新船号 {new_ship_no!r} 含空格，已规整为 {new_no}")
    if conn.execute("SELECT 1 FROM ships WHERE UPPER(REPLACE(ship_no,' ',''))=?",
                    (new_no,)).fetchone():
        return {"error": "新船号已被占用：%s，请换一个序号" % new_no}
    if new_no == norm_ship_no(src_ship_no):
        return {"error": "新船号不能和源船相同"}

    stype = (new_ship_type or "").strip() or (src["ship_type"] or "")
    if not stype:
        stype = resolve_ship_type(new_no) or ""

    # ---- 1) 建 ships 行：业务属性照抄源船，时间节点按选择 ----
    cols = ["ship_no", "ship_type", "destination", "priority", "manual_priority",
            "must_finish_on_time", "allow_delay", "status", "remarks",
            "fixed_workers", "fixed_new", "manpower_mode"]
    vals = [new_no, stype, src["destination"], src["priority"], src["manual_priority"],
            src["must_finish_on_time"], src["allow_delay"], "在制",
            f"复制自 {src_ship_no}", src["fixed_workers"], src["fixed_new"],
            src["manpower_mode"]]
    # 时间节点：要么沿用源船，要么留空（NULL）
    for f in _SHIP_DATE_FIELDS:
        cols.append(f)
        vals.append(src[f] if copy_dates else None)
    # 手填交检日只在沿用日期时才带过来，否则留空（免得不小心继承旧交检日）
    cols.append("manual_inspection")
    vals.append(src["manual_inspection"] if copy_dates else None)
    conn.execute("INSERT INTO ships(%s) VALUES(%s)" % (",".join(cols),
                                                      ",".join("?" * len(cols))), vals)

    # ---- 2) 建 tasks：'template' 按新船型模板展开 / 'clone' 照搬源船并清零 ----
    added, no_template, mode_used = 0, False, task_mode
    if task_mode == "clone":
        src_tasks = conn.execute(
            "SELECT * FROM tasks WHERE ship_no=? ORDER BY id", (src_ship_no,)).fetchall()
        rows = []
        for t in src_tasks:
            rows.append((new_no, stype or t["ship_type"], t["process_code"], t["process_name"],
                         t["team"], t["section"], t["zone"], t["task_type"],
                         t["planned_hours"], 0.0, t["planned_hours"],       # 完成量清零
                         STATUS_TODO, 0.0, t["confidence"], None,
                         t["earliest_day"], t["latest_day"], t["min_w"], t["max_w"],
                         t["critical"], t["milestone"] or "", t["material_ready"],
                         "复制", ""))
        if rows:
            conn.executemany(
                "INSERT INTO tasks(ship_no,ship_type,process_code,process_name,team,section,"
                "zone,task_type,planned_hours,done_hours,remaining_hours,status,progress,"
                "confidence,actual_finish,earliest_day,latest_day,min_w,max_w,critical,"
                "milestone,material_ready,source,remarks) "
                "VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)", rows)
            added = len(rows)
    else:
        tmpl = conn.execute("SELECT * FROM task_templates WHERE ship_type=?",
                            (stype,)).fetchall()
        if not tmpl:
            no_template = True
        rows = []
        for t in tmpl:
            rows.append((new_no, stype, t["process_code"], t["process_name"], t["team"],
                         t["section"], t["zone"], t["task_type"], t["standard_hours"], 0.0,
                         t["standard_hours"], STATUS_TODO, 0.0, 0.6, None, t["earliest_day"],
                         t["latest_day"], 1, 4, 0, t["milestone"] or "", 1, "模板", ""))
        if rows:
            conn.executemany(
                "INSERT INTO tasks(ship_no,ship_type,process_code,process_name,team,section,"
                "zone,task_type,planned_hours,done_hours,remaining_hours,status,progress,"
                "confidence,actual_finish,earliest_day,latest_day,min_w,max_w,critical,"
                "milestone,material_ready,source,remarks) "
                "VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)", rows)
            added = len(rows)

    conn.commit()

    # 刷新新船的统计字段（done_hours / remaining_hours / progress / current_stage）。
    # 不刷新的话界面上新船会显示"剩余 0H"，看起来像复制失败。
    try:
        row = conn.execute(
            "SELECT SUM(planned_hours) p, SUM(done_hours) d, "
            "SUM(CASE WHEN status<>'已完成' THEN remaining_hours ELSE 0 END) r "
            "FROM tasks WHERE ship_no=?", (new_no,)).fetchone()
        p, d, rr = (row["p"] or 0), (row["d"] or 0), (row["r"] or 0)
        stage = conn.execute(
            "SELECT team,SUM(remaining_hours) h FROM tasks WHERE ship_no=? AND status='未开始' "
            "GROUP BY team ORDER BY h DESC LIMIT 1", (new_no,)).fetchone()
        conn.execute("UPDATE ships SET done_hours=?,remaining_hours=?,progress=?,"
                     "current_stage=?,delay_days=-1 WHERE ship_no=?",
                     (round(d, 1), round(rr, 1),
                      round(min(1.0, d / p) if p > 0 else 0.0, 3),
                      (stage["team"] if stage else "未开工"), new_no))
        conn.commit()
    except Exception as e:
        log(f"复制船后刷新统计失败（不影响复制结果）：{e}")

    hours = conn.execute("SELECT COALESCE(SUM(planned_hours),0) h FROM tasks WHERE ship_no=?",
                         (new_no,)).fetchone()["h"]
    log(f"复制船：{src_ship_no} -> {new_no}（船型 {stype}，工序 {added} 道，"
        f"工时 {hours:.1f}H，方式 {task_mode}{'，沿用日期' if copy_dates else '，日期留空'}）")
    return {"ok": True, "ship_no": new_no, "ship_type": stype, "src_ship_no": src_ship_no,
            "tasks": added, "hours": round(float(hours or 0), 1),
            "task_mode": mode_used, "no_template": no_template,
            "copy_dates": bool(copy_dates)}


def api_ship_copy(h, body):
    """
    复制船 / 查询下一个建议船号。

    POST /api/ship/copy
      {act:'next_no', ship_type:'A45S'}                    查该船型的下一个建议船号
      {act:'copy', src:'A45S008', new_ship_no:'A45S009',
       ship_type:'A45S', task_mode:'template', copy_dates:0}   执行复制
      {act:'preview', src:'A45S008', ship_type:'A45S'}     预览会复制什么
    """
    conn = connect()
    try:
        init_db(conn)
        body = body or {}
        act = (body.get("act") or "copy").strip()

        if act == "next_no":
            st = (body.get("ship_type") or "").strip()
            h.json({"ok": True, "ship_type": st, "next_ship_no": next_ship_no(conn, st)})
            return

        if act == "preview":
            src_no = str(body.get("src") or "").strip()
            src = conn.execute("SELECT * FROM ships WHERE ship_no=?", (src_no,)).fetchone()
            if src is None:
                h.json({"error": "源船不存在：%s" % src_no})
                return
            st = (body.get("ship_type") or "").strip() or (src["ship_type"] or "")
            n_task = conn.execute("SELECT COUNT(*) c FROM tasks WHERE ship_no=?",
                                  (src_no,)).fetchone()["c"]
            n_tmpl = conn.execute("SELECT COUNT(*) c FROM task_templates WHERE ship_type=?",
                                  (st,)).fetchone()["c"]
            tmpl_h = conn.execute(
                "SELECT COALESCE(SUM(standard_hours),0) h FROM task_templates WHERE ship_type=?",
                (st,)).fetchone()["h"]
            src_h = conn.execute(
                "SELECT COALESCE(SUM(planned_hours),0) h FROM tasks WHERE ship_no=?",
                (src_no,)).fetchone()["h"]
            h.json({"ok": True, "src_ship_no": src_no, "ship_type": st,
                    "src_tasks": n_task, "src_hours": round(float(src_h or 0), 1),
                    "template_tasks": n_tmpl, "template_hours": round(float(tmpl_h or 0), 1),
                    "suggest_no": next_ship_no(conn, st),
                    "dates": {f: src[f] for f in _SHIP_DATE_FIELDS}})
            return

        res = copy_ship(conn, str(body.get("src") or "").strip(),
                        body.get("new_ship_no"), body.get("ship_type") or "",
                        (body.get("task_mode") or "template").strip(),
                        bool(body.get("copy_dates")))
        h.json(res)
    finally:
        conn.close()


def api_save_zones(h, body):
    conn = connect()
    try:
        for z, cap in (body.get("zones") or {}).items():
            conn.execute("UPDATE zones SET max_teams=? WHERE zone_code=?", (int(cap or 1), z))
        conn.commit()
        h.json({"ok": True})
    finally:
        conn.close()


# ==================================================================
# HTTP 服务
# ==================================================================
class Handler(BaseHTTPRequestHandler):
    def log_message(self, fmt, *args):      # 静默访问日志
        pass

    def json(self, obj):
        data = json.dumps(obj, ensure_ascii=False, default=str).encode("utf-8")
        self.send_response(200)
        self.send_header("Content-Type", "application/json; charset=utf-8")
        self.send_header("Content-Length", str(len(data)))
        self.send_header("Cache-Control", "no-store")
        self.end_headers()
        self.wfile.write(data)

    def html(self, text):
        data = text.encode("utf-8")
        self.send_response(200)
        self.send_header("Content-Type", "text/html; charset=utf-8")
        self.send_header("Content-Length", str(len(data)))
        self.end_headers()
        self.wfile.write(data)

    def do_GET(self):
        u = urlparse(self.path)
        p, q = u.path, parse_qs(u.query)
        try:
            if p == "/precision":
                from precision_web import HTML
                self.html(HTML)
            elif p == "/api/precision":
                from precision_web import handle
                handle(globals(),self,q)
            elif p in ("/", "/index.html"):
                self.html(_load_html())
            elif p == "/api/status":
                api_status(self)
            elif p == "/api/dashboard":
                api_dashboard(self)
            elif p == "/api/quality":
                api_quality(self)
            elif p == "/api/ships":
                api_ships(self)
            elif p == "/api/gantt":
                api_gantt(self, q)
            elif p == "/api/capacity":
                api_capacity(self, q)
            elif p == "/api/risk":
                api_risk(self)
            elif p == "/api/suggest":
                api_suggest(self, q)
            elif p == "/api/settings":
                api_settings(self)
            elif p == "/api/gap":
                api_gap(self)
            elif p == "/api/manpower":
                api_manpower(self, q)
            elif p == "/api/teamplan":
                api_teamplan(self, q)
            elif p == "/api/realtime_files":
                api_realtime_files(self, q)
            elif p == "/api/gongxu":
                api_gongxu(self, q)
            elif p == "/api/expired_ships":
                api_expired_ships(self, q)
            elif p == "/api/keel_gate":
                api_keel_gate(self, q)
            elif p == "/api/estimate":
                api_estimate(self, q)
            elif p == "/api/audit":
                api_audit(self, q)
            elif p == "/api/headcount":
                api_headcount(self, q)
            elif p == "/api/temp_tasks":
                api_temp_tasks(self, q)
            elif p == "/api/hour_rules":
                api_hour_rules(self, q)
            elif p == "/api/scenario_meta":
                api_scenario_meta(self)
            elif p == "/api/logs":
                with _log_lock:
                    self.json(list(aps_engine_logs()))
            else:
                self.send_error(404)
        except Exception as e:
            log(f"GET {p} 出错: {e}")
            try:
                self.json({"error": str(e)})
            except Exception:
                pass

    def do_POST(self):
        u = urlparse(self.path)
        p = u.path
        length = int(self.headers.get("Content-Length") or 0)
        raw = self.rfile.read(length) if length else b"{}"
        try:
            body = json.loads(raw.decode("utf-8"))
        except Exception:
            body = {}
        try:
            if p == "/api/precision":
                from precision_web import handle
                handle(globals(),self,body=body)
            elif p == "/api/copy":
                api_copy(self)
            elif p == "/api/import":
                api_import(self)
            elif p == "/api/schedule":
                api_schedule(self, body)
            elif p == "/api/export":
                api_export(self, body)
            elif p == "/api/scenario":
                api_scenario(self, body)
            elif p == "/api/ship/delay":
                api_ship_delay(self, body)
            elif p == "/api/ships/excluded":
                api_excluded_ships(self, body)
            elif p == "/api/worktime":
                api_worktime(self, body)
            elif p == "/api/holidays":
                api_holidays(self, body)
            elif p == "/api/whatif/add":
                api_whatif_add(self, body)
            elif p == "/api/whatif/worktime":
                api_whatif_worktime(self, body)
            elif p == "/api/manpower":
                api_manpower(self, None, body)
            elif p == "/api/export/teamplan":
                api_export_teamplan(self, body)
            elif p == "/api/upload":
                api_upload(self, body)
            elif p == "/api/realtime_files":
                api_realtime_files(self, None, body)
            elif p == "/api/gongxu":
                api_gongxu(self, None, body)
            elif p == "/api/import/realtime":
                api_import_realtime(self, body)
            elif p == "/api/expired_ships":
                api_expired_ships(self, None, body)
            elif p == "/api/keel_gate":
                api_keel_gate(self, None, body)
            elif p == "/api/estimate":
                api_estimate(self, None, body)
            elif p == "/api/audit":
                api_audit(self, None, body)
            elif p == "/api/headcount":
                api_headcount(self, None, body)
            elif p == "/api/temp_tasks":
                api_temp_tasks(self, None, body)
            elif p == "/api/hour_rules":
                api_hour_rules(self, None, body)
            elif p == "/api/settings/teams":
                api_save_teams(self, body)
            elif p == "/api/settings/ships":
                api_save_ships(self, body)
            elif p == "/api/ships/bulk":
                api_ships_bulk(self, body)
            elif p == "/api/ship/copy":
                api_ship_copy(self, body)
            elif p == "/api/ship/delete":
                api_ship_delete(self, body)
            elif p == "/api/settings/zones":
                api_save_zones(self, body)
            else:
                self.send_error(404)
        except Exception as e:
            log(f"POST {p} 出错: {e}")
            try:
                self.json({"error": str(e)})
            except Exception:
                pass


def pick_port(start_port=DEFAULT_PORT, tries=20):
    """
    选一个空闲端口。

    注意：Windows 上 SO_REUSEADDR 允许两个进程绑定同一个端口，
    只靠 bind 成功判断"端口可用"并不可靠 —— 会出现两个程序都自称占用 8765，
    而浏览器连到的是先启动的那个（表现为"改了代码却不生效"）。
    所以这里先用 connect 探测端口是否真的有人应答，再交给服务器去 bind。
    """
    import socket
    for i in range(tries):
        port = start_port + i
        probe = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        probe.settimeout(0.25)
        try:
            if probe.connect_ex(("127.0.0.1", port)) != 0:
                return port          # 没人应答 -> 认为空闲
        except OSError:
            return port
        finally:
            probe.close()
    return start_port


def main():
    os.chdir(BASE_DIR)
    try:
        conn = connect()
        try:
            init_db(conn)
        finally:
            conn.close()
    except Exception as e:
        _fatal("数据库初始化失败", f"{e}\n\n{traceback.format_exc()}\n"
                                    f"数据库路径：{DB_PATH}\n请确认该文件夹可写。")

    port = pick_port()
    url = f"http://127.0.0.1:{port}"
    _wlog(f"服务启动 {url}")
    # 把地址写到文件，供启动器（.vbs / .bat）读取后自动打开浏览器
    try:
        with open(os.path.join(BASE_DIR, "服务地址.txt"), "w", encoding="utf-8") as f:
            f.write(url)
    except Exception:
        pass
    _p("=" * 62)
    _p("  船舶总装有限产能智能生产排产系统（单机版 · 零依赖）")
    _p("=" * 62)
    _p(f"  [OK] 服务已启动：{url}")
    _p(f"  数据目录：{DATA_DIR}")
    _p(f"  数据库  ：{DB_PATH}")
    _p("")
    _p("  使用步骤：数据导入 → 系统设置（改人数）→ 排产与甘特图")
    _p("")
    _p("  [注意] 请勿关闭本窗口，关闭即停止服务")
    _p("=" * 62)

    try:
        threading.Timer(1.2, lambda: webbrowser.open(url)).start()
    except Exception as e:
        _p(f"  （浏览器未能自动打开，请手动访问 {url}）")
        _wlog(f"自动打开浏览器失败: {e}")

    try:
        ThreadingHTTPServer(("127.0.0.1", port), Handler).serve_forever()
    except KeyboardInterrupt:
        _p("\n已停止服务")
    except Exception as e:
        _fatal("服务运行中断",
               f"{e}\n\n{traceback.format_exc()}\n\n"
               f"常见原因：\n"
               f"  1. 端口 {port} 被安全软件拦截 —— 可换端口后重试\n"
               f"  2. 其他程序占用了该端口")




_planning_lock = threading.RLock()

def run_schedule(conn, start=None, horizon=120, freeze_days=0, scenario=None, note="", mode="normal", keel_gate=None, keel_rule=None):
    from precision_engine import migrate
    if not 1 <= int(horizon) <= 730: raise ValueError('展望天数应为1至730')
    if not 0 <= int(freeze_days) <= 90: raise ValueError('冻结天数应为0至90')
    migrate(conn)
    conn.commit()
    scratch=sqlite3.connect(':memory:');scratch.row_factory=sqlite3.Row
    with _planning_lock:
        conn.backup(scratch)
        try:
            from precision_audit import capture, inspect, save
            inputs=capture(scratch)
            issues=inspect(inputs)
            errors=[x for x in issues if x['level']=='错误']
            if errors:
                raise ValueError('计划输入校验未通过：'+'；'.join(x['ship_no']+'/'+x['process_code']+' '+x['message'] for x in errors[:8]))
            for override in scratch.execute('SELECT * FROM planning_overrides'):
                values=json.loads(override['payload'])
                scratch.execute('UPDATE tasks SET '+','.join(k+'=?' for k in values)+' WHERE ship_no=? AND process_code=?',tuple(values.values())+(override['ship_no'],override['process_code']))
            scratch.commit()
            result=_run_schedule_impl(scratch,start,int(horizon),int(freeze_days),scenario,note,mode,keel_gate,keel_rule)
            if result[0] is None: return result
            version,metrics=result
            metrics['input_issues']=issues
            metrics['engine']='时段排程 2026.10.08-r2'
            scratch.execute('UPDATE schedule_versions SET metrics=? WHERE version_no=?',(json.dumps(metrics,ensure_ascii=False),version))
            save(scratch,version,inputs,dict(start=str(start or dt.date.today()),horizon=int(horizon),freeze_days=int(freeze_days),scenario=scenario or {},mode=mode,keel_gate=keel_gate,keel_rule=keel_rule),issues)
            for ship in scratch.execute('SELECT * FROM ships'):
                scratch.execute('INSERT OR REPLACE INTO schedule_ship_snapshot VALUES(?,?,?)',
                                (version,ship['ship_no'],json.dumps(dict(ship),ensure_ascii=False)))
            scratch.commit()
            # One transaction publishes all related result tables. Inputs stay untouched.
            tables=['ships','calendar','schedule_versions','schedule_result','schedule_ship_snapshot','schedule_input_snapshot']
            with conn:
                for table in tables:
                    cols=[r[1] for r in scratch.execute('PRAGMA table_info('+table+')')]
                    conn.execute('DELETE FROM '+table)
                    conn.executemany('INSERT INTO '+table+' ('+','.join(cols)+') VALUES('+','.join('?' for _ in cols)+')',
                                     [tuple(r) for r in scratch.execute('SELECT * FROM '+table)])
            return result
        finally: scratch.close()

def _isolate_simulation(fn):
    def wrapped(conn,*args,**kwargs):
        with _planning_lock:
            conn.commit()
            # Run the entire scenario group on a private DB. No interim scenario
            # forecast or calendar is visible to the official connection.
            scratch=sqlite3.connect(':memory:');scratch.row_factory=sqlite3.Row
            conn.backup(scratch)
            try:
                result=fn(scratch,*args,**kwargs)
                # Keep draft versions for later comparison/export, never activate.
                existing={r[0] for r in conn.execute('SELECT version_no FROM schedule_versions')}
                with conn:
                    for v in scratch.execute('SELECT * FROM schedule_versions'):
                        if v['version_no'] in existing: continue
                        d=dict(v);d.pop('id');d['is_active']=0
                        conn.execute('INSERT INTO schedule_versions('+','.join(d)+') VALUES('+','.join('?' for _ in d)+')',tuple(d.values()))
                        for table in ('schedule_result','schedule_ship_snapshot','schedule_input_snapshot'):
                            for r in scratch.execute('SELECT * FROM '+table+' WHERE version_no=?',(v['version_no'],)):
                                row=dict(r);row.pop('id',None)
                                conn.execute('INSERT INTO '+table+'('+','.join(row)+') VALUES('+','.join('?' for _ in row)+')',tuple(row.values()))
                return result
            finally:scratch.close()
    return wrapped

for _name in ('simulate_add_workers','simulate_worktime_matrix','simulate_headcount_plan'):
    globals()[_name]=_isolate_simulation(globals()[_name])

# Keep legacy API snapshots complete as well.
_old_snapshot=_ship_snapshot
def _ship_snapshot(conn,ship_no=None):
    out=_old_snapshot(conn,ship_no)
    for r in conn.execute('SELECT * FROM ships'):
        if r['ship_no'] in out: out[r['ship_no']].update({'_full':dict(r)})
    return out

def _restore_official(conn,base_version,snap):
    conn.execute('UPDATE schedule_versions SET is_active=0')
    if base_version: conn.execute('UPDATE schedule_versions SET is_active=1 WHERE version_no=?',(base_version,))
    for sn,s in (snap or {}).items():
        row=s.get('_full')
        if row:
            fields=[k for k in row if k!='ship_no']
            conn.execute('UPDATE ships SET '+','.join(k+'=?' for k in fields)+' WHERE ship_no=?',tuple(row[k] for k in fields)+(sn,))
    conn.commit()

# All HTTP mutations and reads share the scheduler lock, preventing partial UI reads.
def _serialized(fn):
    def call(*a,**kw):
        with _planning_lock:return fn(*a,**kw)
    return call
Handler.do_POST=_serialized(Handler.do_POST)
Handler.do_GET=_serialized(Handler.do_GET)

_old_export_plan=export_plan
def export_plan(conn,version,path=None,extra=None):
    snap=conn.execute('SELECT payload FROM schedule_ship_snapshot WHERE version_no=?',(version,)).fetchall()
    if not snap:return _old_export_plan(conn,version,path,extra)
    conn.commit()
    copy=sqlite3.connect(':memory:');copy.row_factory=sqlite3.Row;conn.backup(copy)
    try:
        for row in snap:
            ship=json.loads(row[0]);fields=[k for k in ship if k!='ship_no']
            copy.execute('UPDATE ships SET '+','.join(k+'=?' for k in fields)+' WHERE ship_no=?',tuple(ship[k] for k in fields)+(ship['ship_no'],))
        copy.commit()
        return _old_export_plan(copy,version,path,extra)
    finally:copy.close()

# 模块命名空间桥接：保持原 IM.xxx() / SC.xxx() 的写法
_IM_NAMES = ['_SHIP_SHEET_RE', 'DESKTOP_CANDIDATES', 'scan_raw_files', 'norm_ship_no', 'merge_duplicate_ships', 'looks_like_ship_file', 'find_realtime_dirs', 'scan_realtime_files', 'classify_excel', 'import_uploaded_file', 'copy_from_desktop', '_TMPL_COLS', 'import_cycle_template', 'GX_GROUPS', 'parse_gongxu_file', 'import_gongxu_hours', '_NODE_FIELDS', 'import_master_plan', '_TEAM_COLS', 'OPTION_SKIP_VALUES', 'import_team_plan', '_RT_COLS', 'import_realtime_progress', '_PLAN_COLS', 'PLAN_DONE_STATUS',
 'import_plan_detail', 'merge_cross_progress', 'expand_tasks', 'sync_ships_from_progress', 'normalize_task_teams', 'sync_actual_progress', 'infer_implicit_completion', 'infer_by_production_day', 'TEAM_DONE_STATUS', 'sync_missing_from_team_plan', 'apply_final_remaining', 'fill_unstarted_teams', 'apply_team_scope', 'apply_option_flag', 'skip_expired_ships', 'restore_expired_ships', 'expired_ships', 'mark_ship_status', 'compute_ship_progress', 'run_full_import']
_SC_NAMES = ['ensure_calendar', 'load_teams', 'team_capacity', 'build_capacity', 'KEEL_RULES', 'keel_status', 'run_schedule', 'suggest_headcount', 'simulate_headcount_plan', 'simulate_add_workers', 'simulate_worktime_matrix', 'risk_rows', 'suggest_actions', 'audit_rows', 'export_audit', 'estimate_finish_by_hours', 'export_estimate', 'team_plan', 'export_team_plan', 'export_plan', '_ship_snapshot', '_restore_official']


IM = types.SimpleNamespace(**{n: globals()[n] for n in _IM_NAMES if n in globals()})
SC = types.SimpleNamespace(**{n: globals()[n] for n in _SC_NAMES if n in globals()})


def _load_html():
    """优先用同目录的 web_ui.html（方便改界面），否则用内嵌的界面。"""
    p = os.path.join(BASE_DIR, "web_ui.html")
    if os.path.isfile(p):
        try:
            with open(p, encoding="utf-8") as f:
                return f.read()
        except Exception:
            pass
    return HTML_UI.replace('<div class="nav" id="nav">', '<div class="nav" id="nav"><div style="padding:12px"><button onclick="window.open(\'/precision\',\'_blank\')">精准计划台</button></div>')

if __name__ == "__main__":
    try:
        main()
    except SystemExit:
        raise
    except Exception as _e:
        _fatal("程序意外终止", f"{_e}\n\n{traceback.format_exc()}")
