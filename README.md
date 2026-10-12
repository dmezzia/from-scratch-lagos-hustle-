# FROM SCRATCH - Lagos Hustle Edition
`npm i && cp .env.example .env.local` (fill Supabase URL + anon key) `&& npm run dev`. Deploy on Vercel with the same env vars.

## 1. Supabase Auth setup
Authentication → Providers → Email: **turn OFF "Confirm email"** (players log in with username + password; the app maps username to `username@fromscratch.game`).

## 2. SQL (run in SQL Editor)
```sql
create table players(id uuid default gen_random_uuid() primary key,auth_id uuid unique default auth.uid(),username text unique not null,capital bigint default 50000,day int default 1,businesses_owned jsonb default '[]',level int default 0,xp int default 0,school_code text,current_street text default 'Ajegunle',state jsonb default '{}',last_sync timestamptz default now(),version int default 0);
create table business_votes(id uuid default gen_random_uuid() primary key,business_name text not null,voter_username text not null,voter_id uuid default auth.uid(),school_code text,created_at timestamptz default now());
create table admins(auth_id uuid primary key);
create table businesses(name text primary key,cost bigint not null,profit bigint not null);
create table streets(name text primary key,level_required int,capital_required bigint,color_theme text,rent_multiplier numeric,profit_multiplier numeric,description text);
alter table players enable row level security;alter table business_votes enable row level security;alter table admins enable row level security;alter table streets enable row level security;alter table businesses enable row level security;
-- players: READ only. All writes come from the `game` Edge Function (service role), so nobody can edit their own score.
create policy "read players" on players for select using(true);
create policy "read votes" on business_votes for select using(true);
create policy "vote as self" on business_votes for insert to authenticated with check(voter_id=auth.uid());
create policy "read streets" on streets for select using(true);
create policy "read businesses" on businesses for select using(true);
create policy "admin self check" on admins for select to authenticated using(auth_id=auth.uid());
insert into businesses values
('POS Business',50000,2000),
('Mama Put / Buka',150000,5000),
('Barbing Salon',200000,6000),
('Mini Mart / Provision',500000,15000),
('Pure Water Factory',80000,3000),
('Bookshop & Stationery',120000,4000),
('Bole & Fish Joint',100000,3333),
('Suya Spot',280000,9333),
('Bakery & Cake',460000,15333),
('Shawarma & Grills',640000,21333),
('Zobo & Smoothie Bar',820000,27333),
('Restaurant/Eatery',1000000,33333),
('Car Wash',100000,3333),
('Laundry',280000,9333),
('Tailoring',460000,15333),
('Phone Accessories',640000,21333),
('Photography',820000,27333),
('Makeup & Gele',1000000,33333),
('Event Decoration',1180000,39333),
('Logistics/Dispatch',1360000,45333),
('Poultry Farm',100000,3333),
('Fish Pond',280000,9333),
('Pig/Goat Farm',460000,15333),
('Crop Farming',640000,21333),
('Snail Farm',820000,27333),
('Soap Making',1000000,33333),
('Palm Oil',1180000,39333),
('Garri Processing',1360000,45333),
('Bead Making',1540000,51333),
('Furniture/Carpentry',1720000,57333),
('Pharmacy/Chemist',600000,20000),
('Tech Hub/Coding',780000,26000),
('Cyber Cafe',960000,32000),
('POS & Crypto Exchange',1140000,38000),
('Real Estate Agency',1320000,44000),
('Car Dealership',1500000,50000),
('Building Materials',1680000,56000),
('Electrical Shop',1860000,62000),
('Fuel/Gas Station',2040000,68000),
('School/Lesson Center',2220000,74000),
('Night Club/Lounge',100000,3333),
('Bet Shop/Viewing Center',280000,9333),
('Hotel/Shortlet',460000,15333),
('Salon/Spa',640000,21333),
('Fitness Gym',820000,27333),
('Music Studio',1000000,33333),
('Boutique',1180000,39333),
('Supermarket',1360000,45333),
('Salon Bus',1540000,51333),
('Transport/Keke',1720000,57333),
('Estate Development',5000000,120000),
('Mega Night Club',8000000,200000),
('Shopping Mall',12000000,300000),
('TV Station',18000000,450000),
('Private Hospital',15000000,380000),
('University',20000000,480000),
('Five-Star Hotel',25000000,600000),
('Oil Depot',30000000,700000),
('Private Bank',40000000,900000),
('Island Mega Estate',50000000,1200000);
insert into streets values('Ajegunle',0,0,'brown',1,1,'Starter'),('Ikeja Computer Village',5,1000000,'blue',2,1.5,'Tech'),('Yaba / Ojuelegba',15,5000000,'yellow',3,2,'Student area'),('Lekki / Ajah',25,15000000,'green',5,3,'Big boy street'),('Victoria Island',40,50000000,'gold',10,5,'Final boss');

-- Votes: server fills username + school from the player row (no spoofing) and enforces 1 vote / business / 24h
create function check_vote() returns trigger language plpgsql security definer set search_path=public as $$ begin
 select username,school_code into new.voter_username,new.school_code from players where auth_id=new.voter_id;
 if new.voter_username is null then raise exception 'Start playing first'; end if;
 if exists(select 1 from business_votes where voter_id=new.voter_id and business_name=new.business_name and created_at>now()-interval '24 hours') then raise exception 'Already voted in 24 hours'; end if;
 return new; end $$;
create trigger vote_once before insert on business_votes for each row execute function check_vote();

```
## 3. Make yourself admin
Sign up in the game as e.g. `oga`, then:
```sql
insert into admins select id from auth.users where email='oga@fromscratch.game';
```
## 4. Deploy the game engine (Edge Function)
```bash
npm i -g supabase
supabase login && supabase link --project-ref YOUR_REF
supabase functions deploy game      # code: supabase/functions/game/index.ts
```
`SUPABASE_URL`, `SUPABASE_ANON_KEY` and `SUPABASE_SERVICE_ROLE_KEY` are provided automatically to Edge Functions. Never put the service-role key in the Next.js app.

## Upgrading an older database
`drop function if exists sync_player(jsonb,text,text); alter table players add column if not exists version int default 0;` and make sure the policies "insert own player"/"update own player" are dropped.

## How anti-cheat works
The browser is now only a screen. The `game` Edge Function runs every day-tick (one per 30 real seconds, max 10 catch-up ticks, only while the player is online), all spending, loans, travel, hiring and the Japa/Landlord choice. It uses a version column so two tabs cannot double-spend. Capital, level and XP in the database can only come from that function. Remaining soft spots: the Danfo mini-game result (max +₦1k per game day) is reported by the browser, and players can open many accounts. For prize money, add email verification and one-account-per-school-seat checks.
