-- Machawi OPS v8 - تطوير متابعة تنفيذ المهام
-- نفّذ مرة واحدة داخل Supabase SQL Editor.

alter table public.role_task_daily_status
  add column if not exists started_at timestamptz,
  add column if not exists reason text,
  add column if not exists note text,
  add column if not exists evidence_path text,
  add column if not exists points_awarded integer not null default 0;

-- تحديث النقاط الحالية
update public.role_task_daily_status
set points_awarded=
  case
    when status='done' then 10
    when status='in_progress' then 4
    else 0
  end
where points_awarded=0;

-- السماح بقراءة الأداء للإدارة ونائب المدير وصاحب الحساب
drop policy if exists role_status_select on public.role_task_daily_status;
create policy role_status_select
on public.role_task_daily_status
for select
to authenticated
using (
  profile_id=auth.uid()
  or public.current_role() in ('admin','manager','deputy')
);

-- فهرس لتسريع التقارير اليومية
create index if not exists idx_role_status_work_date
on public.role_task_daily_status(work_date);

create index if not exists idx_role_status_profile
on public.role_task_daily_status(profile_id);

-- View للتقارير
create or replace view public.role_task_performance_report
with (security_invoker=true)
as
select
  s.work_date,
  s.profile_id,
  p.full_name,
  p.role,
  count(*) as total_updated,
  count(*) filter (where s.status='done') as done_count,
  count(*) filter (where s.status='in_progress') as in_progress_count,
  count(*) filter (where s.status='not_done') as not_done_count,
  coalesce(sum(s.points_awarded),0) as total_points
from public.role_task_daily_status s
join public.profiles p on p.id=s.profile_id
group by s.work_date,s.profile_id,p.full_name,p.role;
