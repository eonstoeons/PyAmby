#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
PyAmby  — cosmic algorithmic ambient player + exporter
Pure Python stdlib. No dependencies. Offline. Forever.
MIT License · 2025 FREEFLOW Project
"""

import tkinter as tk
from tkinter import ttk, scrolledtext, filedialog, messagebox
import threading, queue, os, sys, time, random, re
import wave, tempfile, subprocess, shutil, array
from pathlib import Path

# ═══════════════════════════════════════════════════════════════
#  EMBEDDED SYNTHESIS ENGINE  (exec'd in a private namespace)
# ═══════════════════════════════════════════════════════════════
CORE = r'''
import json, math, os, random, struct, time, wave
from pathlib import Path

SR=44100;INV_SR=1/SR;TAU=math.tau;_PI=math.pi;NYQUIST=SR*.5;CHUNK=SR*10
_sin=math.sin;_cos=math.cos;_exp=math.exp;_tanh=math.tanh
_rand=random.random;_gauss=random.gauss;_uniform=random.uniform
_SIN_N=4096;_SIN_T=[math.sin(i*TAU/_SIN_N) for i in range(_SIN_N)]
def fast_sin(p):
    q=(p%TAU)*(_SIN_N/TAU);i=int(q)&(_SIN_N-1);f=q-int(q)
    return _SIN_T[i]+(_SIN_T[(i+1)&(_SIN_N-1)]-_SIN_T[i])*f
QUALITY={"mobile":{"max_voices":4,"reverb_combs":4,"max_chimes":6,"bytebeat_sr":4000},
         "balanced":{"max_voices":12,"reverb_combs":6,"max_chimes":10,"bytebeat_sr":8000},
         "studio":{"max_voices":32,"reverb_combs":8,"max_chimes":12,"bytebeat_sr":8000}}
_Q=dict(QUALITY["balanced"])
def set_quality(n): global _Q;_Q=dict(QUALITY.get(n,QUALITY["balanced"]))
def clamp(x,lo=-1.,hi=1.): return lo if x<lo else(hi if x>hi else x)
def soft_clip(x,d=1.10): t=_tanh(d);return _tanh(x*d)/t if t else x
def lerp(a,b,t): return a+(b-a)*t
def mtof(n): return 440.*(2.**((n-69.)/12.))
def humanize(t,a=.004): return t+_gauss(0.,a)

COSMIC_OBJECTS=[
    {"name":"Sirius A",      "type":"star",   "temp":9940,   "lum":25.4,    "dist":8.6,    "mass":2.02},
    {"name":"Vega",          "type":"star",   "temp":9600,   "lum":40.1,    "dist":25.0,   "mass":2.14},
    {"name":"Arcturus",      "type":"star",   "temp":4286,   "lum":170.0,   "dist":36.7,   "mass":1.08},
    {"name":"Rigel",         "type":"star",   "temp":12100,  "lum":120000,  "dist":860.0,  "mass":21.0},
    {"name":"Betelgeuse",    "type":"star",   "temp":3500,   "lum":126000,  "dist":700.0,  "mass":16.5},
    {"name":"Antares",       "type":"star",   "temp":3660,   "lum":75900,   "dist":550.0,  "mass":12.4},
    {"name":"Polaris",       "type":"star",   "temp":6015,   "lum":2500,    "dist":433.0,  "mass":5.4},
    {"name":"Deneb",         "type":"star",   "temp":8525,   "lum":196000,  "dist":2600.0, "mass":19.0},
    {"name":"Canopus",       "type":"star",   "temp":7400,   "lum":10700,   "dist":310.0,  "mass":8.5},
    {"name":"Aldebaran",     "type":"star",   "temp":3910,   "lum":518.0,   "dist":65.0,   "mass":1.16},
    {"name":"Spica",         "type":"star",   "temp":25300,  "lum":20500,   "dist":250.0,  "mass":11.4},
    {"name":"Fomalhaut",     "type":"star",   "temp":8590,   "lum":16.6,    "dist":25.1,   "mass":1.92},
    {"name":"Proxima Cen",   "type":"star",   "temp":3042,   "lum":0.0017,  "dist":4.24,   "mass":0.12},
    {"name":"Tau Ceti",      "type":"star",   "temp":5344,   "lum":0.52,    "dist":11.9,   "mass":0.78},
    {"name":"Epsilon Eri",   "type":"star",   "temp":5084,   "lum":0.34,    "dist":10.5,   "mass":0.82},
    {"name":"61 Cygni A",    "type":"star",   "temp":4526,   "lum":0.15,    "dist":11.4,   "mass":0.70},
    {"name":"Barnards Star", "type":"star",   "temp":3134,   "lum":0.0035,  "dist":5.96,   "mass":0.17},
    {"name":"Altair",        "type":"star",   "temp":7550,   "lum":10.6,    "dist":16.7,   "mass":1.79},
    {"name":"Procyon A",     "type":"star",   "temp":6530,   "lum":6.93,    "dist":11.5,   "mass":1.50},
    {"name":"Castor A",      "type":"star",   "temp":8840,   "lum":34.0,    "dist":51.5,   "mass":2.76},
    {"name":"Orion Nebula",  "type":"nebula", "temp":10000,  "lum":400000,  "dist":1344,   "mass":2000},
    {"name":"Helix Nebula",  "type":"nebula", "temp":120000, "lum":300,     "dist":650,    "mass":0.6},
    {"name":"Crab Nebula",   "type":"nebula", "temp":11000,  "lum":100000,  "dist":6500,   "mass":10},
    {"name":"Ring Nebula",   "type":"nebula", "temp":120000, "lum":200,     "dist":2300,   "mass":1.1},
    {"name":"Eagle Nebula",  "type":"nebula", "temp":30000,  "lum":300000,  "dist":7000,   "mass":5000},
    {"name":"Lagoon Nebula", "type":"nebula", "temp":8500,   "lum":280000,  "dist":4100,   "mass":500},
    {"name":"Rosette Nebula","type":"nebula", "temp":35000,  "lum":500000,  "dist":5200,   "mass":10000},
    {"name":"Pillars Creat", "type":"nebula", "temp":8000,   "lum":200000,  "dist":7000,   "mass":900},
    {"name":"Horsehead Neb", "type":"nebula", "temp":50,     "lum":0.001,   "dist":1500,   "mass":100},
    {"name":"Andromeda",     "type":"galaxy", "temp":5000,   "lum":2.6e10,  "dist":2537000,"mass":1.5e12},
    {"name":"Whirlpool",     "type":"galaxy", "temp":7000,   "lum":2.5e10,  "dist":23000000,"mass":1.6e11},
    {"name":"Sombrero",      "type":"galaxy", "temp":5500,   "lum":8e10,    "dist":28000000,"mass":8e11},
    {"name":"Triangulum",    "type":"galaxy", "temp":6000,   "lum":1.4e9,   "dist":2730000,"mass":5e10},
    {"name":"Vela Pulsar",   "type":"remnant","temp":1000000,"lum":11000,   "dist":1000,   "mass":1.86},
    {"name":"Crab Pulsar",   "type":"remnant","temp":2000000,"lum":200000,  "dist":6500,   "mass":1.4},
    {"name":"Cassiopeia A",  "type":"remnant","temp":20000,  "lum":10000,   "dist":11000,  "mass":5},
]

class DCBlocker:
    __slots__=("x","y","R")
    def __init__(self,R=.995): self.x=0.;self.y=0.;self.R=R
    def process(self,v): o=v-self.x+self.R*self.y;self.x=v;self.y=o;return o

class SVF:
    __slots__=("lp","bp","hp","_f","_q")
    def __init__(self,c=1000.,r=0.): self.lp=self.bp=self.hp=0.;self._f=0.;self._q=1.;self.set(c,r)
    def set(self,c,r=None):
        c=min(max(20.,c),NYQUIST*.95);self._f=2.*_sin(_PI*c*INV_SR)
        if r is not None: self._q=1.-min(max(0.,r),.97)
    def lp_process(self,x):
        h=x-self.lp-self._q*self.bp;self.bp+=self._f*h;self.lp+=self._f*self.bp;self.hp=h;return self.lp
    def hp_process(self,x): self.lp_process(x);return self.hp
    def bp_process(self,x): self.lp_process(x);return self.bp

class OnePole:
    __slots__=("a","z")
    def __init__(self,c=1000.): self.z=0.;self.set(c)
    def set(self,c): self.a=1.-_exp(-TAU*min(max(c,1.),NYQUIST*.99)*INV_SR)
    def lp(self,x): self.z+=self.a*(x-self.z);return self.z
    def hp(self,x): return x-self.lp(x)

class ADSR:
    __slots__=("a","d","s","r")
    def __init__(self,a=.01,d=.1,s=.7,r=.3):
        self.a=max(a,.001);self.d=max(d,.001);self.s=s;self.r=max(r,.001)
    def get(self,t,dur):
        if t<0: return 0.
        if t<self.a: return t/self.a
        if t<self.a+self.d: return 1.-(1.-self.s)*((t-self.a)/self.d)
        if t<dur: return self.s
        rt=t-dur;return self.s*(1.-rt/self.r) if rt<self.r else 0.

def osc_saw_blep(p,f):
    t=(p%TAU)/TAU;v=2.*t-1.;dt=f*INV_SR
    if dt<=0: return v
    if t<dt: x=t/dt;v-=x+x-x*x-1.
    elif t>1.-dt: x=(t-1.)/dt;v-=x+x+x*x+1.
    return v
def osc_tri(p): q=(p%TAU)/TAU;return 4.*abs(q-.5)-1.
def osc_sqblep(p,f):
    t=(p%TAU)/TAU;v=1. if t<.5 else -1.;dt=f*INV_SR
    if dt<=0: return v
    if t<dt: x=t/dt;v+=x+x-x*x-1.
    elif t>1.-dt: x=(t-1.)/dt;v-=x+x+x*x+1.
    t2=(t+.5)%1.
    if t2<dt: x=t2/dt;v-=x+x-x*x-1.
    elif t2>1.-dt: x=(t2-1.)/dt;v+=x+x+x*x+1.
    return v

class SuperSaw:
    __slots__=("phases","incs","mix","freq")
    def __init__(self,freq,detune=.30,mix=.82):
        sp=[-.03,-.02,-.01,0.,.01,.02,.03]
        self.phases=[_uniform(0,TAU) for _ in range(7)]
        self.incs=[TAU*freq*(1.+detune*s)*INV_SR for s in sp]
        self.mix=mix/7.;self.freq=freq
    def sample(self,t=0.,env=1.):
        v=0.
        for i in range(7): self.phases[i]+=self.incs[i];v+=osc_saw_blep(self.phases[i],self.freq)
        return v*self.mix

class FMSynth:
    __slots__=("ci","mi","depth","fb","pc","pm","prev")
    def __init__(self,freq,ratio=2.,depth=1.4,feedback=.12):
        self.ci=TAU*freq*INV_SR;self.mi=TAU*freq*ratio*INV_SR
        self.depth=depth;self.fb=feedback;self.pc=0.;self.pm=0.;self.prev=0.
    def sample(self,t=0.,env=1.):
        mod=fast_sin(self.pm+self.prev*self.fb)*self.depth;self.pm+=self.mi
        out=fast_sin(self.pc+mod);self.pc+=self.ci;self.prev=out;return out

class SubSynth:
    __slots__=("phase","inc","wave","filt","env_depth","base_cut","freq")
    def __init__(self,freq,wave="saw",cutoff=2000.,res=.22,env_depth=2500.):
        self.phase=_uniform(0,TAU);self.inc=TAU*freq*INV_SR;self.wave=wave
        self.filt=SVF(cutoff,res);self.env_depth=env_depth;self.base_cut=cutoff;self.freq=freq
    def sample(self,t=0.,env=1.):
        self.phase+=self.inc;w=self.wave
        if w=="saw": v=osc_saw_blep(self.phase,self.freq)
        elif w=="square": v=osc_sqblep(self.phase,self.freq)
        elif w=="tri": v=osc_tri(self.phase)
        else: v=fast_sin(self.phase)
        self.filt.set(min(self.base_cut+self.env_depth*env,NYQUIST*.95))
        return self.filt.lp_process(v)

class KarplusStrong:
    __slots__=("buf","idx","decay","bright")
    def __init__(self,freq,decay=.996,brightness=.55):
        period=max(int(SR/max(freq,24.)),2)
        self.buf=[_uniform(-1,1) for _ in range(period)]
        self.idx=0;self.decay=decay;self.bright=1.-brightness*.7
    def sample(self,t=0.,env=1.):
        L=len(self.buf);i=self.idx%L;j=(i+1)%L;out=self.buf[i]
        self.buf[i]=(self.buf[i]+self.buf[j])*.5*self.bright*self.decay
        self.idx+=1;return out

class Pad:
    __slots__=("phases","incs","lfo","lfo_inc","n")
    def __init__(self,notes,detune=.005):
        ph=[];inc=[]
        for n in notes:
            f=mtof(n)
            for d in(-detune,0.,detune): ph.append(_uniform(0,TAU));inc.append(TAU*f*(1.+d)*INV_SR)
        self.phases=ph;self.incs=inc;self.lfo=_uniform(0,TAU);self.lfo_inc=TAU*.16*INV_SR;self.n=len(ph)
    def sample(self,t=0.,env=1.):
        self.lfo+=self.lfo_inc;mod=.7+.3*(.5+.5*fast_sin(self.lfo))
        v=0.
        for i in range(self.n): self.phases[i]+=self.incs[i];v+=fast_sin(self.phases[i])
        return (v/self.n)*mod if self.n else 0.

class Organ:
    __slots__=("phase","inc")
    def __init__(self,freq): self.phase=0.;self.inc=TAU*freq*INV_SR
    def sample(self,t=0.,env=1.):
        self.phase+=self.inc;p=self.phase
        return .60*fast_sin(p)+.22*fast_sin(p*2)+.10*fast_sin(p*3)+.06*fast_sin(p*4)

class FluteSynth:
    __slots__=("phase","inc","vib_phase","vib_inc","breath_lp","freq")
    def __init__(self,freq):
        self.phase=_uniform(0,TAU);self.inc=TAU*freq*INV_SR
        self.vib_phase=0.;self.vib_inc=TAU*5.5*INV_SR;self.breath_lp=OnePole(4000.);self.freq=freq
    def sample(self,t=0.,env=1.):
        self.vib_phase+=self.vib_inc;vib=fast_sin(self.vib_phase)*.012*min(t*4,1.)
        self.phase+=self.inc*(1.+vib);breath=self.breath_lp.lp(_uniform(-1,1))*.08
        return fast_sin(self.phase)*.85+breath

class Kick808:
    __slots__=("punch","decay","tone")
    def __init__(self,punch=1.,decay=.40,tone=50.): self.punch=punch;self.decay=decay;self.tone=tone
    def sample(self,t=0.,env=1.):
        if t<0: return 0.
        return soft_clip((_sin(TAU*(self.tone+190.*self.punch*_exp(-30.*t))*t)*_exp(-t/max(self.decay,.001))
                          +_exp(-150.*t)*.40*self.punch)*.95,1.25)

class Snare909:
    __slots__=("tone","noise_amt","decay","hpf")
    def __init__(self,tone=195.,noise_amt=.62,decay=.18):
        self.tone=tone;self.noise_amt=noise_amt;self.decay=decay;self.hpf=SVF(2200.,.04)
    def sample(self,t=0.,env=1.):
        if t<0: return 0.
        te=_exp(-18.*t);ne=_exp(-t/max(self.decay,.001))
        return soft_clip((_sin(TAU*self.tone*t)*te*(1.-self.noise_amt)+
                          self.hpf.hp_process(_uniform(-1,1))*ne*self.noise_amt)*.82,1.35)

class HiHat:
    __slots__=("decay","hpf","phases","incs")
    RATIOS=(1.,1.342,1.2312,1.6532,1.9523,2.1523)
    def __init__(self,open_hat=False,decay=.045):
        self.decay=.22 if open_hat else decay;self.hpf=SVF(7800.,.05)
        self.phases=[0.]*6;self.incs=[TAU*420.*r*INV_SR for r in self.RATIOS]
    def sample(self,t=0.,env=1.):
        if t<0: return 0.
        amp=_exp(-t/max(self.decay,.001));v=0.
        for i in range(6): self.phases[i]+=self.incs[i];v+=1. if(self.phases[i]%TAU)<_PI else -1.
        return self.hpf.hp_process(v/6.)*amp*.48

class PinkNoise:
    __slots__=("b",)
    def __init__(self): self.b=[0.]*6
    def sample(self):
        b=self.b;w=_uniform(-1,1)
        b[0]=.99886*b[0]+w*.0555179;b[1]=.99332*b[1]+w*.0750759
        b[2]=.96900*b[2]+w*.1538520;b[3]=.86650*b[3]+w*.3104856
        b[4]=.55000*b[4]+w*.5329522;b[5]=-.7616*b[5]-w*.0168980
        return(b[0]+b[1]+b[2]+b[3]+b[4]+b[5]+w*.5362)*.11

class BrownNoise:
    __slots__=("z",)
    def __init__(self): self.z=0.
    def sample(self): self.z=.98*self.z+.03*_uniform(-1,1);return clamp(self.z)

class Wind:
    __slots__=("pink","pink2","lfo_b","lfo_g","inc_b","inc_g","lp1","hp","bp","intensity")
    def __init__(self,intensity=.55):
        self.pink=PinkNoise();self.pink2=PinkNoise()
        self.lfo_b=_uniform(0,TAU);self.lfo_g=_uniform(0,TAU)
        self.inc_b=TAU*.12*INV_SR;self.inc_g=TAU*.025*INV_SR
        self.lp1=OnePole(1200.);self.hp=OnePole(60.);self.bp=SVF(800.,.15);self.intensity=intensity
    def sample_stereo(self,t=0.):
        self.lfo_b+=self.inc_b;self.lfo_g+=self.inc_g
        breath=.5+.5*fast_sin(self.lfo_b);gust=max(0,fast_sin(self.lfo_g)**3)
        mod=.15+.45*breath+.35*gust;v1=self.pink.sample();v2=self.pink2.sample()
        cut=600.+1400.*gust+400.*breath;self.lp1.set(cut);self.bp.set(cut*.7,.12+.15*gust)
        l=self.hp.hp(self.lp1.lp(v1)+self.bp.bp_process(v1)*.3*gust)
        r=self.hp.hp(self.lp1.lp(v2)+self.bp.bp_process(v2)*.25*gust)
        s=mod*self.intensity;return l*s,r*s*.93

class Rain:
    __slots__=("bed","slp","mlp","hlp","hp","intensity")
    def __init__(self,intensity=.60):
        self.bed=PinkNoise();self.slp=OnePole(8000.);self.mlp=OnePole(3500.)
        self.hlp=OnePole(1800.);self.hp=OnePole(200.);self.intensity=intensity
    def sample_stereo(self,t=0.):
        i=self.intensity;b=self.hp.hp(self.bed.sample())*.08*i
        s=self.slp.lp(_uniform(-1,1))*_uniform(.02,.06)*i if _rand()<i*.06 else 0.
        m=self.mlp.lp(_uniform(-1,1))*_uniform(.08,.18)*i if _rand()<i*.012 else 0.
        h=self.hlp.lp(_uniform(-1,1))*_uniform(.15,.30)*i if _rand()<i*.003 else 0.
        mono=b+s+m+h;pan=_uniform(-.4,.4);pr=(pan+1.)*_PI*.25
        return mono*_cos(pr),mono*_sin(pr)

class Ocean:
    __slots__=("p1","p2","p3","foam","lfo1","lfo2","lfo3","inc1","inc2","inc3",
               "lp1","lp2","lp3","hp","flp","ct","ca","cd","intensity")
    def __init__(self,intensity=.65):
        self.p1=PinkNoise();self.p2=PinkNoise();self.p3=PinkNoise();self.foam=PinkNoise()
        self.lfo1=_uniform(0,TAU);self.lfo2=_uniform(0,TAU);self.lfo3=_uniform(0,TAU)
        self.inc1=TAU*.045*INV_SR;self.inc2=TAU*.11*INV_SR;self.inc3=TAU*.28*INV_SR
        self.lp1=OnePole(400.);self.lp2=OnePole(900.);self.lp3=OnePole(2200.)
        self.hp=OnePole(25.);self.flp=OnePole(6000.)
        self.ct=-1.;self.ca=0.;self.cd=0.;self.intensity=intensity
    def sample_stereo(self,t=0.):
        self.lfo1+=self.inc1;self.lfo2+=self.inc2;self.lfo3+=self.inc3
        sw=max(0,fast_sin(self.lfo1))**1.5;w2=.5+.5*fast_sin(self.lfo2);rp=.5+.5*fast_sin(self.lfo3)
        v=self.hp.hp(self.lp1.lp(self.p1.sample())*sw*.55+self.lp2.lp(self.p2.sample())*w2*.28+self.lp3.lp(self.p3.sample())*rp*.12)
        if self.ct<0 and sw>.85 and _rand()<.02: self.ct=0.;self.ca=_uniform(.3,.7);self.cd=_uniform(1.5,3.5)
        cr=0.
        if self.ct>=0:
            self.ct+=INV_SR
            if self.ct<self.cd: cr=self.flp.lp(self.foam.sample())*_exp(-self.ct*2./self.cd)*self.ca*.4
            else: self.ct=-1.
        mono=(v+cr)*self.intensity*.75;sp=.5+.15*fast_sin(self.lfo1*.3)
        return mono*sp,mono*(1.-sp)

class WaterStream:
    __slots__=("pink","hpf","lpf","blp","intensity")
    def __init__(self,intensity=.55):
        self.pink=PinkNoise();self.hpf=OnePole(500.);self.lpf=OnePole(4500.)
        self.blp=OnePole(2800.);self.intensity=intensity
    def sample_stereo(self,t=0.):
        v=self.hpf.hp(self.lpf.lp(self.pink.sample()))*.18
        b=self.blp.lp(_uniform(-1,1))*_uniform(.1,.35) if _rand()<self.intensity*.008 else 0.
        sp=_uniform(.15,.4) if _rand()<self.intensity*.0015 else 0.
        mono=(v+b*.3+sp*.15)*self.intensity;pan=_uniform(-.5,.5);pr=(pan+1)*_PI*.25
        return mono*_cos(pr),mono*_sin(pr)

class Fire:
    __slots__=("rlp","clp","chp","php","ebp","lfo","lfo_inc","intensity")
    def __init__(self,intensity=.55):
        self.rlp=OnePole(250.);self.clp=OnePole(2500.);self.chp=OnePole(400.)
        self.php=OnePole(1500.);self.ebp=SVF(4500.,.3);self.lfo=_uniform(0,TAU);self.lfo_inc=TAU*.07*INV_SR
        self.intensity=intensity
    def sample_stereo(self,t=0.):
        self.lfo+=self.lfo_inc;br=.6+.4*(.5+.5*fast_sin(self.lfo))
        roar=self.rlp.lp(_uniform(-1,1))*.12*br
        cr=self.chp.hp(self.clp.lp(_uniform(-1,1)))*_uniform(.2,.5) if _rand()<self.intensity*.015 else 0.
        pop=self.php.hp(_uniform(-1,1))*_uniform(.3,.7) if _rand()<self.intensity*.004 else 0.
        mono=(roar+cr*.25+pop*.15)*self.intensity;pan=_uniform(-.3,.3);pr=(pan+1)*_PI*.25
        return mono*_cos(pr),mono*_sin(pr)

class Thunder:
    __slots__=("active","stage","st","amp","dist","lp1","lp2","lp3","hp","intensity","cool")
    def __init__(self,intensity=.70):
        self.active=False;self.stage=0;self.st=0.;self.amp=0.;self.dist=.5
        self.lp1=OnePole(80.);self.lp2=OnePole(250.);self.lp3=OnePole(500.)
        self.hp=OnePole(20.);self.intensity=intensity;self.cool=0.
    def sample_stereo(self,t=0.):
        if self.cool>0: self.cool-=INV_SR
        if not self.active:
            if self.cool<=0 and _rand()<self.intensity*.00003:
                self.active=True;self.stage=0;self.st=0.
                self.amp=_uniform(.5,1.);self.dist=_uniform(.2,1.);self.cool=_uniform(5.,15.)
                self.lp1.set(60+80*(1-self.dist));self.lp2.set(150+200*(1-self.dist))
            return 0.,0.
        self.st+=INV_SR;v=0.
        if self.stage==0:
            if self.st<.05*(1+self.dist): v=self.lp3.lp(_uniform(-1,1))*_exp(-self.st*30)*self.amp*.6
            else: self.stage=1;self.st=0.
        elif self.stage==1:
            dur=1.5+1.5*self.dist
            if self.st<dur:
                env=_exp(-self.st*1.5/dur)
                v=(self.lp1.lp(_uniform(-1,1))*.5+self.lp2.lp(_uniform(-1,1))*.35*_exp(-self.st*3))*env*self.amp
            else: self.stage=2;self.st=0.
        elif self.stage==2:
            tail=2.+4.*self.dist
            if self.st<tail: v=self.lp1.lp(_uniform(-1,1))*_exp(-self.st*2./tail)*self.amp*.25
            else: self.active=False;return 0.,0.
        v=self.hp.hp(v)*self.intensity*.65;w=.4*(1-self.dist)+.05;return v*(.5+w),v*(.5-w)

class SingingBowl:
    __slots__=("partials","decay","t","ns","amp","intensity")
    def __init__(self,intensity=.60):
        self.partials=[];self.decay=0.;self.t=0.;self.ns=0.;self.amp=0.;self.intensity=intensity;self._strike()
    def _strike(self):
        base=random.choice([180.,220.,260.,310.,370.,440.])
        self.partials=[]
        for r,b in zip([1.,2.71,4.77,7.03,9.43],[0.,.7,1.2,.5,.9]):
            f=base*r
            if f<NYQUIST*.9: self.partials.append([_uniform(0,TAU),TAU*f*INV_SR,TAU*b*INV_SR,1./(r*r*.3+1.)])
        self.decay=_uniform(5.,12.);self.t=0.;self.amp=_uniform(.5,1.);self.ns=_uniform(6.,18.)
    def sample(self,t=0.):
        self.t+=INV_SR
        if self.t>self.ns: self._strike()
        env=_exp(-self.t*2./max(self.decay,.01))
        if env<.001: return 0.
        v=0.
        for p in self.partials:
            p[0]+=p[1];v+=fast_sin(p[0])*p[3]*(.85+.15*fast_sin(p[0]*.0001+p[2]*self.t*SR))
        return v*env*self.amp*self.intensity*.16

class WindChimes:
    __slots__=("density","chimes","intensity")
    def __init__(self,density=.40,intensity=.50): self.density=density;self.chimes=[];self.intensity=intensity
    def sample_stereo(self,t=0.):
        if len(self.chimes)<_Q["max_chimes"] and _rand()<self.density*.0003:
            f=random.choice([1200,1580,1890,2100,2640,3150,3520])
            self.chimes.append([0.,float(f),_uniform(3,8),0.,_uniform(.3,1.),_uniform(-.5,.5)])
        l=0.;r=0.;alive=[]
        for ch in self.chimes:
            ch[0]+=TAU*ch[1]*INV_SR;ch[3]+=INV_SR;env=_exp(-ch[3]/ch[2])
            if env<.001: continue
            alive.append(ch)
            tone=fast_sin(ch[0])*.6+fast_sin(ch[0]*2.76)*.25+fast_sin(ch[0]*5.4)*.12
            v=tone*env*ch[4]*self.intensity*.12;pr=(ch[5]+1)*_PI*.25;l+=v*_cos(pr);r+=v*_sin(pr)
        self.chimes=alive;return l,r

ENTITY_PROFILES={
    "glacier":    {"coherence":.99,"collapse":.0005,"entangle":.95,"pulse":.005,"centroid":30},
    "mountain":   {"coherence":.99,"collapse":.001, "entangle":.90,"pulse":.01, "centroid":60},
    "canyon":     {"coherence":.97,"collapse":.003, "entangle":.80,"pulse":.02, "centroid":80},
    "nebula":     {"coherence":.60,"collapse":.12,  "entangle":.85,"pulse":2.,  "centroid":3500},
    "pulsar":     {"coherence":.55,"collapse":.20,  "entangle":.25,"pulse":150.,"centroid":8000},
    "stellar_wind":{"coherence":.70,"collapse":.08, "entangle":.50,"pulse":.8,  "centroid":2200},
    "dark_matter":{"coherence":.99,"collapse":.0001,"entangle":.99,"pulse":.001,"centroid":10},
    "cosmic_void":{"coherence":1., "collapse":.0001,"entangle":1., "pulse":.0005,"centroid":5},
}

class CosmicEntity:
    __slots__=("profile","lp","hp","bp","pp","pi","clfo","ci","ct","noise","intensity","eb","eix")
    def __init__(self,pname="mountain",intensity=.35):
        p=ENTITY_PROFILES.get(pname,ENTITY_PROFILES["mountain"]);self.profile=p;c=p["centroid"]
        self.lp=OnePole(c*2);self.hp=OnePole(max(c*.1,10));self.bp=SVF(c,.3)
        self.pp=0.;self.pi=TAU*p["pulse"]*INV_SR;self.clfo=_uniform(0,TAU);self.ci=TAU*.07*INV_SR
        self.ct=-1.;self.noise=PinkNoise();self.intensity=intensity;self.eb=[0.]*256;self.eix=0
    def sample_stereo(self,t=0.):
        p=self.profile;self.clfo+=self.ci;coh=p["coherence"]*(.8+.2*fast_sin(self.clfo))
        if self.ct<0 and _rand()<p["collapse"]*INV_SR*.1: self.ct=0.
        cv=0.
        if self.ct>=0:
            self.ct+=INV_SR;dur=.02+p["collapse"]*.5
            if self.ct<dur: cv=_uniform(-1,1)*_exp(-self.ct/dur*3)
            else: self.ct=-1.
        self.pp+=self.pi;pulse=fast_sin(self.pp)*coh*.4;raw=self.noise.sample()
        bl=self.bp.bp_process(self.lp.lp(raw))*.5+pulse*.5+cv*.3
        ent=p["entangle"];self.eb[self.eix%256]=bl;delayed=self.eb[(self.eix-int(44*ent))%256]
        br=lerp(self.hp.hp(self.noise.sample())*.5+pulse*.5,delayed,ent)+cv*.25
        self.eix+=1;s=self.intensity*(.8+.2*coh);return bl*s,br*s

class FreeflowMode:
    __slots__=("ff","rng","pt","pd","cur","tgt","pink","lp","hp",
               "la","lb","lc","ia","ib","ic","pad","padt","paddur","entity","bp","bi","dcl","dcr")
    def __init__(self,ff=.5,seed=0):
        self.ff=ff;self.rng=random.Random(seed);self.pink=PinkNoise()
        self.lp=OnePole(800.);self.hp=OnePole(60.)
        self.la=_uniform(0,TAU);self.lb=_uniform(0,TAU);self.lc=_uniform(0,TAU)
        r=self.rng;self.ia=TAU*r.uniform(.05,.25)*INV_SR;self.ib=TAU*r.uniform(.02,.12)*INV_SR
        self.ic=TAU*r.uniform(.01,.06)*INV_SR;self.pt=0.;self.pd=r.uniform(8.,20.+ff*40)
        self.cur=self._rs();self.tgt=self._rs();self.pad=None;self.padt=0.;self.paddur=0.
        ep=random.choice(list(ENTITY_PROFILES.keys()));self.entity=CosmicEntity(ep,intensity=.12)
        root=r.choice([36,40,43,45,48]);self.bp=0.;self.bi=TAU*mtof(root-12)*INV_SR
        self.dcl=DCBlocker();self.dcr=DCBlocker()
    def _rs(self):
        r=self.rng
        return{"na":r.uniform(0.,.4*self.ff),"ta":r.uniform(.3,.9),"ba":r.uniform(0.,.35),
               "ea":r.uniform(0.,.25*self.ff),"br":r.uniform(200,4000),
               "notes":[r.randint(48,72) for _ in range(r.randint(2,4))]}
    def sample_stereo(self,t=0.):
        self.pt+=INV_SR;alpha=min(self.pt/max(self.pd,1.),1.)
        def iv(k): a=self.cur[k];b=self.tgt[k];return lerp(a,b,alpha) if isinstance(a,float) else a
        na=iv("na");ta=iv("ta");ba=iv("ba");ea=iv("ea");br=iv("br")
        if alpha>=1.:
            self.cur=self.tgt;self.tgt=self._rs();self.pt=0.;self.pd=self.rng.uniform(8.,20.+self.ff*40)
            ep=random.choice(list(ENTITY_PROFILES.keys()));self.entity=CosmicEntity(ep,intensity=.12)
        self.la+=self.ia;self.lb+=self.ib;self.lc+=self.ic
        self.lp.set(br*(.8+.4*fast_sin(self.la)))
        nl=self.hp.hp(self.lp.lp(self.pink.sample()))*na
        self.padt+=INV_SR
        if self.pad is None or self.padt>self.paddur:
            self.pad=Pad(self.tgt["notes"],detune=.003+self.ff*.008)
            self.paddur=self.rng.uniform(4.,16.);self.padt=0.
        tonal=self.pad.sample(t)*ta*(.5+.5*fast_sin(self.lb))
        self.bp+=self.bi;bass=fast_sin(self.bp)*ba*.5*(.3+.7*fast_sin(self.lc))
        el,er=self.entity.sample_stereo(t)
        l=nl*.7+tonal*.5+bass+el*ea;r=nl*1.3+tonal*.5+bass+er*ea
        return self.dcl.process(l),self.dcr.process(r)

class Reverb:
    CD=(.02257,.02391,.02641,.02743,.02999,.03119,.03371,.03571)
    AP=(.0050,.0017,.00051)
    __slots__=("combs","ci","cfb","aps","ai","lps","damp","mix")
    def __init__(self,size=.9,damp=.45,mix=.30):
        nc=_Q["reverb_combs"];self.combs=[([0.]*max(int(SR*d*size),2)) for d in self.CD[:nc]]
        self.ci=[0]*nc;self.cfb=.84;self.aps=[([0.]*max(int(SR*d),2)) for d in self.AP]
        self.ai=[0]*3;self.lps=[0.]*nc;self.damp=damp;self.mix=mix
    def process(self,x):
        out=0.
        for i,buf in enumerate(self.combs):
            idx=self.ci[i]%len(buf);val=buf[idx]
            self.lps[i]=val*(1.-self.damp)+self.lps[i]*self.damp
            buf[idx]=x+self.lps[i]*self.cfb;self.ci[i]+=1;out+=val
        out/=len(self.combs)
        for i,buf in enumerate(self.aps):
            idx=self.ai[i]%len(buf);bv=buf[idx]
            buf[idx]=out+bv*.5;self.ai[i]+=1;out=bv-out*.5
        return x*(1.-self.mix)+out*self.mix

class Delay:
    __slots__=("bl","br","il","ir","fb","mix")
    def __init__(self,tl=.375,tr=.25,fb=.35,mix=.25):
        self.bl=[0.]*(int(SR*tl)+2);self.br=[0.]*(int(SR*tr)+2);self.il=0;self.ir=0;self.fb=fb;self.mix=mix
    def process(self,l,r):
        il=self.il%len(self.bl);ir=self.ir%len(self.br)
        dl=self.bl[il];dr=self.br[ir];self.bl[il]=l+dl*self.fb;self.br[ir]=r+dr*self.fb
        self.il+=1;self.ir+=1;return l+dl*self.mix,r+dr*self.mix

class Chorus:
    __slots__=("buf","idx","ph1","ph2","inc1","inc2","depth","mix")
    def __init__(self,rate=1.1,depth=.003,mix=.28):
        self.buf=[0.]*(int(SR*.04)+2);self.idx=0;self.ph1=0.;self.ph2=_PI/3.
        self.inc1=TAU*rate*INV_SR;self.inc2=TAU*rate*1.07*INV_SR;self.depth=depth*SR;self.mix=mix
    def process(self,x):
        L=len(self.buf);self.buf[self.idx%L]=x;self.idx+=1
        self.ph1+=self.inc1;self.ph2+=self.inc2;m=self.mix
        def _tap(ph):
            off=self.depth*(1.+fast_sin(ph))*.5;ri=self.idx-int(off);f=off-int(off)
            a=self.buf[ri%L];b=self.buf[(ri-1)%L];return a+(b-a)*f
        return x*(1-m)+_tap(self.ph1)*m,x*(1-m)+_tap(self.ph2)*m

class VinylTexture:
    __slots__=("pink","rlp","crackle")
    def __init__(self): self.pink=PinkNoise();self.rlp=OnePole(70.);self.crackle=0.
    def sample(self):
        h=self.pink.sample()*.010;r=self.rlp.lp(_uniform(-1,1))*.005
        c=0.
        if self.crackle>.001: self.crackle*=.88;c=_uniform(-1,1)*self.crackle
        elif _rand()<.00025: self.crackle=.28
        return h+r+c

SCALES={"major":[0,2,4,5,7,9,11],"minor":[0,2,3,5,7,8,10],"dorian":[0,2,3,5,7,9,10],
        "phrygian":[0,1,3,5,7,8,10],"lydian":[0,2,4,6,7,9,11],"mixolydian":[0,2,4,5,7,9,10],
        "pentatonic":[0,2,4,7,9],"minor_pentatonic":[0,3,5,7,10],"blues":[0,3,5,6,7,10],
        "whole_tone":[0,2,4,6,8,10],"harmonic_minor":[0,2,3,5,7,8,11]}
CHORDS={"maj":[0,4,7],"min":[0,3,7],"maj7":[0,4,7,11],"min7":[0,3,7,10],
        "sus2":[0,2,7],"sus4":[0,5,7],"dim":[0,3,6],"aug":[0,4,8],"add9":[0,4,7,14]}
PROGRESSIONS={
    "ambient":    [(0,"sus2"),(4,"sus2"),(3,"maj7"),(0,"sus2")],
    "dark_ambient":[(0,"min"),(1,"min"),(4,"min"),(3,"maj")],
    "synthwave":  [(0,"min"),(3,"maj"),(4,"min"),(6,"maj")],
    "lo_fi":      [(0,"maj7"),(1,"min7"),(2,"min7"),(4,"maj7")],
    "sacred":     [(0,"sus2"),(5,"sus4"),(3,"maj7"),(0,"sus2")],
    "cosmic":     [(0,"sus2"),(4,"add9"),(3,"sus4"),(2,"min7"),(0,"sus2")],
    "drone":      [(0,"sus2")],
    "meditation": [(0,"sus2"),(4,"sus2")],
    "bellscape":  [(0,"sus2"),(4,"sus2"),(0,"sus2")],
    "experimental":[(0,"sus2"),(1,"min"),(2,"min"),(4,"maj"),(0,"sus2")],
}
GENRES={
    "ambient":{"bpm":(60,76),"scale":"pentatonic","key":48,"prog":"ambient","density":.20,"drums":False,
        "bass_wave":"sine","bass_oct":-2,"bass_cut":300,"bass_res":.08,"lead":"fm","lead_oct":1,
        "lead_detune":.06,"lead_vol":.26,"pad":True,"pad_vol":.62,"reverb_size":1.20,
        "reverb_damp":.38,"reverb_mix":.54,"delay_time":.50,"delay_fb":.40,"delay_mix":.40,
        "chorus_rate":.65,"chorus_depth":.004,"duration":(200,380)},
    "dark_ambient":{"bpm":(52,66),"scale":"phrygian","key":45,"prog":"dark_ambient","density":.14,"drums":False,
        "bass_wave":"sine","bass_oct":-2,"bass_cut":220,"bass_res":.18,"lead":"fm","lead_oct":0,
        "lead_detune":.04,"lead_vol":.22,"pad":True,"pad_vol":.72,"reverb_size":1.42,
        "reverb_damp":.62,"reverb_mix":.64,"delay_time":.65,"delay_fb":.42,"delay_mix":.40,
        "chorus_rate":.38,"chorus_depth":.005,"duration":(260,520)},
    "meditation":{"bpm":(50,62),"scale":"pentatonic","key":48,"prog":"meditation","density":.10,"drums":False,
        "bass_wave":"sine","bass_oct":-2,"bass_cut":180,"bass_res":.03,"lead":"fm","lead_oct":1,
        "lead_detune":.015,"lead_vol":.18,"pad":True,"pad_vol":.70,"reverb_size":1.55,
        "reverb_damp":.58,"reverb_mix":.66,"delay_time":.75,"delay_fb":.42,"delay_mix":.42,
        "chorus_rate":.42,"chorus_depth":.003,"duration":(300,720)},
    "drone":{"bpm":(40,52),"scale":"minor","key":36,"prog":"drone","density":.05,"drums":False,
        "bass_wave":"sine","bass_oct":-2,"bass_cut":150,"bass_res":.02,"lead":"fm","lead_oct":0,
        "lead_detune":.12,"lead_vol":.14,"pad":True,"pad_vol":.82,"reverb_size":1.60,
        "reverb_damp":.70,"reverb_mix":.70,"delay_time":1.,"delay_fb":.46,"delay_mix":.45,
        "chorus_rate":.25,"chorus_depth":.006,"duration":(300,600)},
    "bellscape":{"bpm":(68,84),"scale":"pentatonic","key":60,"prog":"bellscape","density":.26,"drums":False,
        "bass_wave":"sine","bass_oct":-1,"bass_cut":420,"bass_res":.06,"lead":"karplus","lead_oct":1,
        "lead_detune":0.,"lead_vol":.44,"pad":True,"pad_vol":.28,"reverb_size":1.10,
        "reverb_damp":.28,"reverb_mix":.52,"delay_time":.375,"delay_fb":.30,"delay_mix":.34,
        "chorus_rate":.82,"chorus_depth":.002,"duration":(150,300)},
    "sacred":{"bpm":(45,58),"scale":"pentatonic","key":48,"prog":"sacred","density":.16,"drums":False,
        "bass_wave":"sine","bass_oct":-2,"bass_cut":240,"bass_res":.06,"lead":"karplus","lead_oct":1,
        "lead_detune":0.,"lead_vol":.40,"pad":True,"pad_vol":.60,"reverb_size":1.35,
        "reverb_damp":.45,"reverb_mix":.58,"delay_time":.60,"delay_fb":.36,"delay_mix":.34,
        "chorus_rate":.55,"chorus_depth":.003,"duration":(240,480)},
    "cosmic":{"bpm":(42,58),"scale":"whole_tone","key":40,"prog":"cosmic","density":.12,"drums":False,
        "bass_wave":"sine","bass_oct":-2,"bass_cut":200,"bass_res":.05,"lead":"fm","lead_oct":1,
        "lead_detune":.08,"lead_vol":.20,"pad":True,"pad_vol":.75,"reverb_size":1.60,
        "reverb_damp":.50,"reverb_mix":.68,"delay_time":.80,"delay_fb":.45,"delay_mix":.44,
        "chorus_rate":.30,"chorus_depth":.005,"duration":(300,600)},
    "synthwave":{"bpm":(94,112),"scale":"minor","key":43,"prog":"synthwave","density":.54,"drums":True,
        "bass_wave":"saw","bass_oct":-1,"bass_cut":1300,"bass_res":.30,"lead":"supersaw","lead_oct":1,
        "lead_detune":.30,"lead_vol":.36,"pad":True,"pad_vol":.42,"reverb_size":.88,
        "reverb_damp":.32,"reverb_mix":.34,"delay_time":.375,"delay_fb":.34,"delay_mix":.28,
        "chorus_rate":1.0,"chorus_depth":.003,"duration":(180,320)},
    "lo_fi":{"bpm":(74,90),"scale":"dorian","key":48,"prog":"lo_fi","density":.36,"drums":True,
        "bass_wave":"tri","bass_oct":-1,"bass_cut":580,"bass_res":.14,"lead":"fm","lead_oct":1,
        "lead_detune":.018,"lead_vol":.24,"pad":True,"pad_vol":.30,"reverb_size":.72,
        "reverb_damp":.56,"reverb_mix":.30,"delay_time":.333,"delay_fb":.28,"delay_mix":.20,
        "chorus_rate":1.05,"chorus_depth":.003,"duration":(120,260)},
    "experimental":{"bpm":(55,110),"scale":"blues","key":42,"prog":"experimental","density":.42,"drums":False,
        "bass_wave":"pulse","bass_oct":-1,"bass_cut":1400,"bass_res":.28,"lead":"karplus","lead_oct":1,
        "lead_detune":.08,"lead_vol":.34,"pad":True,"pad_vol":.30,"reverb_size":.95,
        "reverb_damp":.28,"reverb_mix":.42,"delay_time":.333,"delay_fb":.40,"delay_mix":.34,
        "chorus_rate":1.4,"chorus_depth":.004,"duration":(120,300)},
}
PURE_MODES={"pure_nature","pure_wind","pure_rain","pure_ocean","pure_water","pure_fire",
            "pure_storm","pure_white","pure_pink","pure_brown","pure_bowls","pure_chimes",
            "pure_theta","pure_alpha","pure_delta"}
SPECIAL_MODES={"freeflow","entity"}
ALL_MODES=set(GENRES.keys())|PURE_MODES|SPECIAL_MODES

MOOD_PRESETS={"peaceful":(.20,.25,.80,.55,.70,1.,.20,.70,.20,.40,.50,.20,.60,.30),
              "ethereal": (.18,.30,.50,.60,.90,1.5,.25,.80,.15,.35,.60,.35,.70,.35),
              "melancholic":(.25,.35,.65,.35,.65,.9,.30,.60,.25,.20,.55,.30,.55,.40),
              "mysterious":(.22,.40,.40,.35,.80,1.4,.40,.75,.35,.30,.65,.55,.50,.45),
              "somber":   (.15,.20,.55,.20,.75,.85,.15,.65,.10,.25,.40,.15,.70,.30)}

def cosmic_to_params(obj):
    temp=obj.get("temp",6000);lum=obj.get("lum",1.);dist=obj.get("dist",100.);mass=obj.get("mass",1.)
    if temp<4000:   mode="dark_ambient";mood="melancholic"
    elif temp<6000: mode="ambient";     mood="peaceful"
    elif temp<7500: mode="meditation";  mood="ethereal"
    elif temp<12000:mode="bellscape";   mood="mysterious"
    elif temp<30000:mode="cosmic";      mood="ethereal"
    else:           mode="experimental";mood="somber"
    import math as m
    bpm=int(max(38,min(120,62-m.log10(max(mass,.001))*4)))
    flags=set()
    if temp<4000: flags.add("sparse")
    if lum>10000: flags.add("wide")
    return{"mode":mode,"mood":mood,"bpm":bpm,"flags":flags,"_cosmic_source":obj.get("name","?")}

def apply_cosmic_seed(params,objname=None):
    obj=None
    if objname:
        nl=objname.lower()
        for o in COSMIC_OBJECTS:
            if nl in o["name"].lower(): obj=o;break
    if obj is None: obj=random.choice(COSMIC_OBJECTS)
    mapped=cosmic_to_params(obj)
    if not params.get("_mode_explicit"): params["mode"]=mapped["mode"];params["mood"]=mapped["mood"]
    if not params.get("bpm"): params["bpm"]=mapped["bpm"]
    params["flags"]=set(params.get("flags",set()))|mapped["flags"]
    params["_cosmic_source"]=mapped["_cosmic_source"]
    return params

def build_scale(root,sn,octaves=3):
    pat=SCALES.get(sn,SCALES["minor"])
    return[root+o*12+i for o in range(octaves) for i in pat if 0<=root+o*12+i<=127]
def build_chord(root,cn): return[root+i for i in CHORDS.get(cn,CHORDS["min"])]
def euclidean_rhythm(steps,pulses):
    if pulses<=0: return[False]*steps
    if pulses>=steps: return[True]*steps
    pat=[];bucket=0
    for _ in range(steps):
        bucket+=pulses
        if bucket>=steps: pat.append(True);bucket-=steps
        else: pat.append(False)
    return pat
def energy_curve(t,dur):
    p=t/max(dur,1.)
    if p<.08: return(p/.08)*.30
    if p<.25: return .30+((p-.08)/.17)*.70
    if p<.70: return 1.
    if p<.80: return 1.-((p-.70)/.10)*.40
    if p<.90: return .60+((p-.80)/.10)*.40
    return max(0.,1.-(p-.90)/.10)

def apply_flags(cfg,flags):
    cfg=dict(cfg)
    if "nodrums" in flags: cfg["drums"]=False
    if "drums"   in flags: cfg["drums"]=True
    if "warm"    in flags: cfg["bass_cut"]=int(cfg["bass_cut"]*.7);cfg["reverb_damp"]=max(cfg["reverb_damp"],.55)
    if "cold"    in flags: cfg["bass_cut"]=int(cfg["bass_cut"]*1.3);cfg["reverb_damp"]=min(cfg["reverb_damp"],.30)
    if "bright"  in flags: cfg["lead_vol"]=min(.8,cfg["lead_vol"]+.05)
    if "sparse"  in flags: cfg["density"]*=.55
    if "dense"   in flags: cfg["density"]=min(1,cfg["density"]*1.4)
    return cfg
def apply_mood_preset(cfg,mood):
    if mood not in MOOD_PRESETS: return cfg
    cfg=dict(cfg)
    (density,_,warmth,_,wetness,_,_,_,_,_,_,_,_,intensity)=MOOD_PRESETS[mood]
    cfg["density"]=lerp(cfg.get("density",.3),density,.6)
    cfg["pad_vol"]=lerp(cfg.get("pad_vol",.5),intensity,.4)
    cfg["reverb_mix"]=lerp(cfg.get("reverb_mix",.4),wetness*.8,.5)
    cfg["reverb_damp"]=lerp(cfg.get("reverb_damp",.5),1.-warmth,.5)
    return cfg

class Event:
    __slots__=("time","duration","engine","vol","pan","env","is_sub","is_drum")
    def __init__(self,time,dur,engine,vol=.5,pan=0.,env=None,is_sub=False,is_drum=False):
        self.time=time;self.duration=dur;self.engine=engine
        self.vol=vol;self.pan=pan;self.env=env or ADSR(.01,.1,.7,.3)
        self.is_sub=is_sub;self.is_drum=is_drum

def make_events(mode,rng,bpm=None,root=None,scale_name=None,duration=None,flags=None,mood=None):
    flags=flags or set()
    genre=apply_flags(GENRES.get(mode,GENRES["ambient"]),flags)
    if mood: genre=apply_mood_preset(genre,mood)
    bpm=bpm or rng.randint(*genre["bpm"]);beat=60./bpm;bar=beat*4
    root=root if root is not None else genre["key"]
    scale_name=scale_name or genre["scale"]
    duration=duration or rng.randint(*genre["duration"])
    total_bars=max(1,int(duration/bar))
    scale=build_scale(root,scale_name,3);bass_scale=build_scale(root+genre["bass_oct"]*12,scale_name,2)
    prog=PROGRESSIONS.get(genre["prog"],PROGRESSIONS["ambient"])
    events=[];density=genre["density"];mv=_Q["max_voices"]
    if genre["pad"]:
        sb=rng.choice([2,4,4,8])
        for bn in range(0,total_bars,sb):
            en=energy_curve(bn*bar,duration)
            if en<.12: continue
            rd,ct=prog[bn%len(prog)];notes=build_chord(scale[rd%len(scale)]+12,ct)
            t=bn*bar;dur=min(bar*rng.choice([2,4,4,8]),max(.5,duration-t))
            events.append(Event(t,dur,Pad(notes,detune=rng.uniform(.003,.008)),
                vol=genre["pad_vol"]*(.35+.65*en),pan=rng.uniform(-.25,.25),
                env=ADSR(rng.uniform(.5,2),rng.uniform(.4,1),rng.uniform(.65,.92),rng.uniform(1.2,3.2))))
    for bn in range(total_bars):
        en=energy_curve(bn*bar,duration);rd,_=prog[bn%len(prog)];br=bass_scale[rd%len(bass_scale)]
        bpat=euclidean_rhythm(16,rng.choice([4,6,8]))
        for step in range(16):
            if bpat[step] and rng.random()<density*(.4+.6*en):
                t=humanize(bn*bar+step*beat*.25,.004);nd=beat*rng.choice([.5,1,1,2])
                freq=mtof(br+rng.choice([0,0,0,12]))
                events.append(Event(t,nd,SubSynth(freq,wave=genre["bass_wave"],cutoff=genre["bass_cut"],
                    res=genre["bass_res"],env_depth=genre["bass_cut"]*2.2),vol=.56,pan=0.,
                    env=ADSR(.006,.12,.60,.12),is_sub=True))
    lc=0
    for bn in range(total_bars):
        if lc>=mv*4: break
        en=energy_curve(bn*bar,duration)
        if rng.random()>density*(.30+.70*en): continue
        rd,ct=prog[bn%len(prog)];cr=scale[rd%len(scale)]+genre["lead_oct"]*12
        nn=rng.randint(2,7)
        for n in range(nn):
            t=humanize(bn*bar+n*bar/max(1,nn),.006);nd=bar/max(1,nn)*rng.uniform(.5,.9)
            if t>=duration: continue
            note=(rng.choice(build_chord(cr,ct)) if rng.random()<.60 else rng.choice(scale)+genre["lead_oct"]*12)
            freq=mtof(note);lt=genre["lead"];is_sub=False
            if lt=="supersaw":  synth=SuperSaw(freq,detune=genre["lead_detune"],mix=rng.uniform(.60,.90))
            elif lt=="fm":      synth=FMSynth(freq,ratio=rng.choice([1,2,3,4,.5]),depth=rng.uniform(.6,2.8),feedback=rng.uniform(0,.25))
            elif lt=="karplus": synth=KarplusStrong(freq,decay=rng.uniform(.992,.998),brightness=rng.uniform(.3,.8))
            elif lt=="organ":   synth=Organ(freq)
            elif lt=="flute":   synth=FluteSynth(freq)
            else:               synth=SubSynth(freq,wave="saw",cutoff=3000,res=.20,env_depth=2200);is_sub=True
            events.append(Event(t,nd,synth,vol=genre["lead_vol"]*(.50+.50*en),pan=rng.uniform(-.45,.45),
                env=ADSR(rng.uniform(.01,.05),rng.uniform(.08,.25),rng.uniform(.30,.75),rng.uniform(.08,.35)),is_sub=is_sub))
            lc+=1
    if genre["drums"]:
        kp=euclidean_rhythm(16,4);hp=euclidean_rhythm(16,rng.choice([8,10,12,14]))
        for bn in range(total_bars):
            en=energy_curve(bn*bar,duration)
            for step in range(16):
                t=humanize(bn*bar+step*beat*.25,.003)
                if t>=duration: continue
                if kp[step] and rng.random()<(.65+.35*en):
                    events.append(Event(t,.5,Kick808(punch=rng.uniform(.78,1.1),decay=rng.uniform(.30,.48),
                        tone=rng.choice([46,50,54])),vol=.78,pan=0,env=ADSR(.001,.01,1,.10),is_drum=True))
                if step in(4,12) and rng.random()<(.45+.55*en):
                    events.append(Event(t,.3,Snare909(tone=rng.uniform(180,220),noise_amt=rng.uniform(.50,.70),
                        decay=rng.uniform(.14,.22)),vol=.55,pan=rng.uniform(-.10,.10),env=ADSR(.001,.01,1,.04),is_drum=True))
                if hp[step]:
                    oh=(step%8==6) and rng.random()<.25
                    events.append(Event(t,.14,HiHat(open_hat=oh,decay=.22 if oh else rng.uniform(.025,.060)),
                        vol=(.28 if step%2==0 else .18)*(.35+.65*en),pan=rng.uniform(-.35,.35),
                        env=ADSR(.001,.008,1,.02),is_drum=True))
    if not events:
        events.append(Event(0,max(1,duration*.9),Pad(build_chord(root+12,"sus2"),detune=.004),
            vol=max(.22,genre.get("pad_vol",.3)),pan=0,env=ADSR(.02,.20,.85,.50)))
        events.append(Event(0,max(.8,duration*.75),SubSynth(mtof(root-12),wave="sine",
            cutoff=max(120,genre.get("bass_cut",200)),res=.05,env_depth=max(200,genre.get("bass_cut",200))),
            vol=.30,pan=0,env=ADSR(.02,.15,.70,.25),is_sub=True))
    return events,duration,bpm,root,scale_name

def pure_mix(mode):
    mx={"wind":0,"rain":0,"ocean":0,"water":0,"fire":0,"thunder":0,
        "white":0,"pink":0,"brown":0,"binaural":0,"bowls":0,"chimes":0}
    if mode=="pure_nature":  mx.update({"wind":.30,"rain":.25,"ocean":.20,"water":.20,"fire":.05})
    elif mode=="pure_wind":  mx["wind"]=1.
    elif mode=="pure_rain":  mx["rain"]=1.
    elif mode=="pure_ocean": mx["ocean"]=1.
    elif mode=="pure_water": mx["water"]=1.
    elif mode=="pure_fire":  mx["fire"]=1.
    elif mode=="pure_storm": mx.update({"wind":.45,"rain":.65,"thunder":.70})
    elif mode=="pure_white": mx["white"]=1.
    elif mode=="pure_pink":  mx["pink"]=1.
    elif mode=="pure_brown": mx["brown"]=1.
    elif mode=="pure_bowls": mx["bowls"]=1.
    elif mode=="pure_chimes":mx.update({"chimes":.80,"wind":.15})
    elif mode=="pure_theta": mx["binaural"]=.8
    elif mode=="pure_alpha": mx["binaural"]=.8
    elif mode=="pure_delta": mx["binaural"]=.8
    return mx

def master_proc(l,r,dcl,dcr,width,gain,drive):
    mid=(l+r)*.5;side=(l-r)*.5*width;l=mid+side;r=mid-side
    l=dcl.process(soft_clip(l,drive)*gain);r=dcr.process(soft_clip(r,drive)*gain)
    return clamp(l,-.999,.999),clamp(r,-.999,.999)

def render_music(path,events,dur,mode,flags,vol,progress_cb):
    genre=apply_flags(GENRES.get(mode,GENRES["ambient"]),flags)
    rl=Reverb(size=genre["reverb_size"],damp=genre["reverb_damp"],mix=genre["reverb_mix"])
    rr=Reverb(size=genre["reverb_size"]*1.07,damp=min(.95,genre["reverb_damp"]*1.04),mix=genre["reverb_mix"])
    dl=Delay(tl=genre["delay_time"],tr=genre["delay_time"]*.75,fb=genre["delay_fb"],mix=genre["delay_mix"])
    ch=Chorus(rate=genre["chorus_rate"],depth=genre["chorus_depth"],mix=.26)
    vt=VinylTexture() if "vinyl" in flags else None
    dcl=DCBlocker();dcr=DCBlocker()
    bl=SingingBowl(.30) if "bowls" in flags else None
    wc=WindChimes(.25,.20) if "chimes" in flags else None
    lw=Wind(.15) if "nature" in flags else None
    lr=Rain(.20) if "nature" in flags else None
    events=sorted(events,key=lambda e:e.time)
    total=int(dur*SR)+SR;ev_ptr=0;active=[]
    width=1.12
    if "wide" in flags: width=min(2.,width*1.35)
    if "narrow" in flags: width=max(.3,width*.65)
    mg=.88*max(0.,min(1.,vol));md=1.10;pm=-1
    with wave.open(str(path),"wb") as w:
        w.setnchannels(2);w.setsampwidth(2);w.setframerate(SR)
        for cs in range(0,total,CHUNK):
            ce=min(cs+CHUNK,total);frames=bytearray((ce-cs)*4);fi=0
            for i in range(cs,ce):
                t=i*INV_SR
                while ev_ptr<len(events) and events[ev_ptr].time<=t+.005:
                    active.append(events[ev_ptr]);ev_ptr+=1
                sl=0.;sr_=0.;still=[]
                for ev in active:
                    lt=t-ev.time
                    if lt>ev.duration+ev.env.r+1.2: continue
                    still.append(ev);env_v=ev.env.get(lt,ev.duration)
                    if env_v<.00005: continue
                    v=(ev.engine.sample(lt,env_v) if ev.is_sub else ev.engine.sample(lt))*env_v*ev.vol
                    pr=(ev.pan+1)*_PI*.25;sl+=v*_cos(pr);sr_+=v*_sin(pr)
                active=still
                if bl:  bv=bl.sample(t);sl+=bv;sr_+=bv
                if wc:  cl2,cr2=wc.sample_stereo(t);sl+=cl2;sr_+=cr2
                if lw:  wl,wr=lw.sample_stereo(t);sl+=wl;sr_+=wr
                if lr:  rl2,rr2=lr.sample_stereo(t);sl+=rl2;sr_+=rr2
                cl,cr=ch.process(sl);cl=rl.process(cl);cr=rr.process(cr)
                cl,cr=dl.process(cl,cr)
                if vt: tx=vt.sample();cl+=tx;cr+=tx*(1+_uniform(-.20,.20))
                cl,cr=master_proc(cl,cr,dcl,dcr,width,mg,md)
                struct.pack_into("<hh",frames,fi,int(cl*32767),int(cr*32767));fi+=4
            w.writeframes(frames);pct=int((ce/total)*100)
            if pct!=pm:
                if progress_cb: progress_cb(pct)
                pm=pct

def render_pure(path,mode,dur,seed,flags,vol,progress_cb):
    mx=pure_mix(mode);total=int(dur*SR)
    wind  =Wind(mx["wind"])        if mx["wind"]>0   else None
    rain  =Rain(mx["rain"])        if mx["rain"]>0   else None
    ocean =Ocean(mx["ocean"])      if mx["ocean"]>0  else None
    water =WaterStream(mx["water"])if mx["water"]>0  else None
    fire  =Fire(mx["fire"])        if mx["fire"]>0   else None
    thund =Thunder(mx["thunder"])  if mx["thunder"]>0 else None
    pink  =PinkNoise()             if mx["pink"]>0   else None
    brown =BrownNoise()            if mx["brown"]>0  else None
    bowls =SingingBowl(mx["bowls"])if mx["bowls"]>0  else None
    chimes=WindChimes(mx["chimes"]*.8,mx["chimes"]*.5) if mx["chimes"]>0 else None
    bb_beat={"pure_theta":4.,"pure_alpha":10.,"pure_delta":2.}.get(mode,0.)
    bbp=0.
    rl=Reverb(size=1.1,damp=.50,mix=.28);rr=Reverb(size=1.16,damp=.54,mix=.28)
    dcl=DCBlocker();dcr=DCBlocker()
    width=1.12
    if "wide" in flags: width=min(2.,width*1.35)
    mg=.88*max(0.,min(1.,vol));md=1.10;pm=-1
    with wave.open(str(path),"wb") as w:
        w.setnchannels(2);w.setsampwidth(2);w.setframerate(SR)
        for cs in range(0,total,CHUNK):
            ce=min(cs+CHUNK,total);frames=bytearray((ce-cs)*4);fi=0
            for i in range(cs,ce):
                t=i*INV_SR;l=0.;r=0.
                if wind:  wl,wr=wind.sample_stereo(t);l+=wl;r+=wr
                if rain:  rl2,rr2=rain.sample_stereo(t);l+=rl2;r+=rr2
                if ocean: ol,or_=ocean.sample_stereo(t);l+=ol;r+=or_
                if water: wsl,wsr=water.sample_stereo(t);l+=wsl;r+=wsr
                if fire:  fl,fr=fire.sample_stereo(t);l+=fl;r+=fr
                if thund: tl,tr=thund.sample_stereo(t);l+=tl;r+=tr
                if mx["white"]>0: v=_uniform(-1,1)*mx["white"]*.25;l+=v;r+=v
                if pink:  v=pink.sample()*mx["pink"]*.30;l+=v;r+=v
                if brown: v=brown.sample()*mx["brown"]*.38;l+=v;r+=v
                if bowls: v=bowls.sample(t);l+=v;r+=v
                if chimes:cl2,cr2=chimes.sample_stereo(t);l+=cl2;r+=cr2
                if bb_beat>0:
                    l+=fast_sin(bbp)*mx["binaural"]*.25;r+=fast_sin(bbp+TAU*bb_beat*t)*mx["binaural"]*.25
                    bbp+=TAU*200.*INV_SR
                l=rl.process(l);r=rr.process(r)
                l,r=master_proc(l,r,dcl,dcr,width,mg,md)
                struct.pack_into("<hh",frames,fi,int(l*32767),int(r*32767));fi+=4
            w.writeframes(frames);pct=int((ce/total)*100)
            if pct!=pm:
                if progress_cb: progress_cb(pct)
                pm=pct

def render_special(path,mode,dur,seed,flags,vol,progress_cb,entity_profile="mountain",ff=.5):
    total=int(dur*SR);dcl=DCBlocker();dcr=DCBlocker()
    rl=Reverb(size=1.2,damp=.45,mix=.40);rr=Reverb(size=1.28,damp=.48,mix=.40)
    width=1.12
    if "wide" in flags: width=min(2.,width*1.35)
    mg=.88*max(0.,min(1.,vol));md=1.10;pm=-1
    gen=CosmicEntity(entity_profile,intensity=.55) if mode=="entity" else FreeflowMode(ff=ff,seed=seed)
    with wave.open(str(path),"wb") as w:
        w.setnchannels(2);w.setsampwidth(2);w.setframerate(SR)
        for cs in range(0,total,CHUNK):
            ce=min(cs+CHUNK,total);frames=bytearray((ce-cs)*4);fi=0
            for i in range(cs,ce):
                t=i*INV_SR;l,r=gen.sample_stereo(t)
                l=rl.process(l);r=rr.process(r)
                l,r=master_proc(l,r,dcl,dcr,width,mg,md)
                struct.pack_into("<hh",frames,fi,int(l*32767),int(r*32767));fi+=4
            w.writeframes(frames);pct=int((ce/total)*100)
            if pct!=pm:
                if progress_cb: progress_cb(pct)
                pm=pct

def render(params,progress_cb=None):
    mode  =params.get("mode","ambient")
    seed  =params.get("seed",int(time.time()*1000)%(2**32))
    flags =set(params.get("flags",[]))
    dur   =params.get("duration",180)
    bpm   =params.get("bpm",None)
    root  =params.get("root",None)
    scale =params.get("scale",None)
    mood  =params.get("mood",None)
    ep    =params.get("entity_profile","mountain")
    ff    =params.get("freeflow",.5)
    qual  =params.get("quality","balanced")
    vol   =params.get("volume",1.)
    out   =Path(params.get("out_dir",str(Path.home()/"Documents"/"PyAmby"))).expanduser()
    out.mkdir(parents=True,exist_ok=True)
    set_quality(qual);random.seed(seed)
    ts=time.strftime("%Y%m%d_%H%M%S");wav=out/f"pyamby_{mode}_{seed}_{ts}.wav"
    if mode in PURE_MODES:
        render_pure(wav,mode,dur,seed,flags,vol,progress_cb)
    elif mode in SPECIAL_MODES:
        render_special(wav,mode,dur,seed,flags,vol,progress_cb,entity_profile=ep,ff=ff)
    else:
        rng=random.Random(seed)
        events,dur,bpm_v,root_v,scale_v=make_events(mode,rng,bpm=bpm,root=root,scale_name=scale,
                                                     duration=dur,flags=flags,mood=mood)
        render_music(wav,events,dur,mode,flags,vol,progress_cb)
    return wav
'''

# ═══════════════════════════════════════════════════════════════
#  AUDIO PLAYBACK  (cross-platform, no dependencies)
# ═══════════════════════════════════════════════════════════════

def _wav_dur(path):
    try:
        with wave.open(str(path),'r') as w:
            return w.getnframes()/w.getframerate()
    except Exception:
        return 90.

def play_wav_blocking(path, stop_ev=None):
    path=str(path); dur=_wav_dur(path); proc=None
    def _wait(p):
        start=time.time()
        while p.poll() is None:
            if stop_ev and stop_ev.is_set():
                try: p.terminate()
                except: pass
                return
            if time.time()-start>dur+8:
                try: p.terminate()
                except: pass
                return
            time.sleep(.08)
    def _tw():
        start=time.time()
        while time.time()-start<dur:
            if stop_ev and stop_ev.is_set(): return
            time.sleep(.08)
    try:
        if sys.platform=='win32':
            try:
                import winsound as _ws
                _ws.PlaySound(path,_ws.SND_FILENAME|_ws.SND_ASYNC)
                start=time.time()
                while time.time()-start<dur+1:
                    if stop_ev and stop_ev.is_set():
                        try: _ws.PlaySound(None,_ws.SND_PURGE)
                        except: pass
                        return
                    time.sleep(.08)
                return
            except Exception: pass
            if shutil.which('powershell'):
                proc=subprocess.Popen(['powershell','-c',
                    f'(New-Object System.Media.SoundPlayer "{path}").PlaySync()'],
                    stdout=subprocess.DEVNULL,stderr=subprocess.DEVNULL)
                _wait(proc);return
        elif sys.platform=='darwin':
            if shutil.which('afplay'):
                proc=subprocess.Popen(['afplay',path],stdout=subprocess.DEVNULL,stderr=subprocess.DEVNULL)
                _wait(proc);return
        for player,extra in[('aplay',[]),('paplay',[]),('play',[]),
            ('ffplay',['-nodisp','-autoexit']),('mpv',['--no-video','--really-quiet']),
            ('cvlc',['--play-and-exit','--intf','dummy']),('mplayer',['-really-quiet']),
            ('termux-media-player',['play'])]:
            if shutil.which(player):
                proc=subprocess.Popen([player]+extra+[path],stdout=subprocess.DEVNULL,stderr=subprocess.DEVNULL)
                _wait(proc);return
        _tw()
    except Exception: _tw()

def detect_player():
    if sys.platform=='win32':
        try: import winsound;return 'winsound'
        except: pass
        if shutil.which('powershell'): return 'powershell'
    elif sys.platform=='darwin':
        if shutil.which('afplay'): return 'afplay'
    for p in['aplay','paplay','play','ffplay','mpv','cvlc','mplayer','termux-media-player']:
        if shutil.which(p): return p
    return None

# ═══════════════════════════════════════════════════════════════
#  INFINITE ENGINE
# ═══════════════════════════════════════════════════════════════

_MUSIC_POOL=['ambient','ambient','ambient','bellscape','bellscape',
             'cosmic','cosmic','meditation','meditation','sacred','drone']
_MOOD_POOL=['ethereal','ethereal','peaceful','peaceful','mysterious','melancholic',None,None,None]

class Engine:
    def __init__(self):
        self._playing=False; self._stop=threading.Event()
        self._q=queue.Queue(maxsize=2)
        self._scb=None; self._pcb=None; self._playcb=None
        self._ns={}; self._loaded=False; self._tmp=None
        self._vol=1.0; self._fixed=None

    def _load(self):
        if not self._loaded:
            ns={'__name__':'pyamby_core','__file__':''}
            exec(CORE,ns)
            self._ns=ns; self._loaded=True
        if self._tmp is None:
            self._tmp=Path(tempfile.gettempdir())/'pyamby_chunks'
            self._tmp.mkdir(exist_ok=True)

    def set_cbs(self,s=None,p=None,play=None): self._scb=s;self._pcb=p;self._playcb=play
    def set_vol(self,v): self._vol=max(0.,min(1.,float(v)))

    def start(self,fixed=None):
        if self._playing: self.stop()
        self._load(); self._fixed=fixed
        self._playing=True; self._stop.clear()
        threading.Thread(target=self._gen,daemon=True,name='pa-gen').start()
        threading.Thread(target=self._play,daemon=True,name='pa-play').start()

    def stop(self):
        self._playing=False; self._stop.set()
        for _ in range(self._q.qsize()+2):
            try:
                p=self._q.get_nowait()
                try: os.unlink(p)
                except: pass
            except queue.Empty: break

    def _emit(self,m):
        if self._scb: self._scb(m)
    def _prog(self,p):
        if self._pcb: self._pcb(int(p))

    def _gen(self):
        render=self._ns['render']; OBJS=self._ns['COSMIC_OBJECTS']
        ac=self._ns['apply_cosmic_seed']; is_nat=self._fixed and self._fixed.startswith('pure_')
        while self._playing:
            mode=self._fixed if self._fixed else random.choice(_MUSIC_POOL)
            seed=random.randint(0,2**32-1)
            if is_nat:
                self._emit(f"generating  ·  {mode.replace('pure_','').replace('_',' ')}")
            else:
                obj=random.choice(OBJS); mood=random.choice(_MOOD_POOL)
                self._emit(f"generating  ·  {obj['name']}  →  {mode}")
            self._prog(0)
            params={'mode':mode,'duration':90,'seed':seed,'flags':set(),'quality':'balanced',
                    'mp3':False,'out_dir':str(self._tmp),'_mode_explicit':True,'volume':self._vol}
            if not is_nat:
                try:
                    params=ac(params,obj['name']); params['mode']=mode
                    if mood: params['mood']=mood
                except: pass
            params['volume']=self._vol
            try:
                path=render(params,progress_cb=self._prog)
            except Exception as e:
                self._emit(f"error — {e}"); time.sleep(2); continue
            if not self._playing:
                try: os.unlink(path)
                except: pass
                break
            try: self._q.put(str(path),timeout=120)
            except queue.Full:
                try: os.unlink(path)
                except: pass

    def _play(self):
        while self._playing:
            try: path=self._q.get(timeout=1)
            except queue.Empty: continue
            if not self._playing:
                try: os.unlink(path)
                except: pass
                break
            mode_disp=Path(path).stem.split('_')[1]
            self._prog(100); self._emit(f"playing  ·  {mode_disp.replace('pure','').strip()}")
            if self._playcb: self._playcb(mode_disp)
            play_wav_blocking(path,self._stop); self._stop.clear()
            try: os.unlink(path)
            except: pass

# ═══════════════════════════════════════════════════════════════
#  NOTE HELPER
# ═══════════════════════════════════════════════════════════════

def note_to_midi(s):
    nm={"C":0,"C#":1,"D":2,"D#":3,"E":4,"F":5,"F#":6,"G":7,"G#":8,"A":9,"A#":10,"B":11}
    m=re.match(r'([A-Ga-g][#b]?)(\d)',s.strip())
    if m:
        n=m.group(1).upper().replace('B','b')
        if n.endswith('b'): n=n[:-1];base=nm.get(n,0)-1
        else: base=nm.get(n,0)
        return (int(m.group(2))+1)*12+base
    return 60

# ═══════════════════════════════════════════════════════════════
#  COLOURS
# ═══════════════════════════════════════════════════════════════
BG='#07070f'; BG2='#0e0e1c'; BG3='#12122a'; PANEL='#0b0b18'
ACC='#6b5ce7'; ACC2='#8b7cf0'; TEAL='#00d4aa'; RED='#ff4466'
DIM='#2a2a44'; MUTED='#505070'; BRIGHT='#d0d0f0'; STAR='#b0a0ff'
NAT='#44cc88'; NAT2='#1a7a44'; SEP='#1a1a2e'

ALL_MODES_LIST = [
    'ambient','dark_ambient','meditation','drone','bellscape','sacred',
    'cosmic','synthwave','lo_fi','experimental',
    'pure_nature','pure_wind','pure_rain','pure_ocean','pure_water',
    'pure_fire','pure_storm','pure_bowls','pure_chimes',
    'pure_white','pure_pink','pure_brown',
    'pure_theta','pure_alpha','pure_delta',
    'freeflow','entity',
]

# ═══════════════════════════════════════════════════════════════
#  PyAmby  GUI
# ═══════════════════════════════════════════════════════════════

class PyAmby:
    def __init__(self):
        self.root=tk.Tk()
        self.root.title('PyAmby')
        self.root.configure(bg=BG)
        self.root.geometry('620x510')
        self.root.minsize(520,420)

        self.engine=Engine()
        self.engine.set_cbs(s=self._on_status,p=self._on_prog,play=self._on_play)
        self._mode=None        # active playback mode key or None
        self._nat_btns={}
        self._no_player=detect_player() is None
        self._exp_rendering=False
        self._exp_last_path=None

        self._vol_var=tk.DoubleVar(value=0.85)
        self.engine.set_vol(0.85)

        self._build()

    # ── main window ─────────────────────────────────────────────

    def _build(self):
        # ── title strip
        hdr=tk.Frame(self.root,bg=BG,pady=7)
        hdr.pack(fill=tk.X,padx=14)
        tk.Label(hdr,text='PyAmby',font=('Courier',18,'bold'),fg=ACC,bg=BG).pack(side=tk.LEFT)
        tk.Label(hdr,text='cosmic sound · offline · forever',
                 font=('Courier',7),fg=DIM,bg=BG).pack(side=tk.LEFT,padx=10,pady=4)

        tk.Frame(self.root,bg=SEP,height=1).pack(fill=tk.X)

        # ── notebook
        sty=ttk.Style(); sty.theme_use('default')
        sty.configure('P.TNotebook',background=BG,borderwidth=0,tabmargins=0)
        sty.configure('P.TNotebook.Tab',background=BG2,foreground=MUTED,
                      font=('Courier',8,'bold'),padding=[14,5],borderwidth=0)
        sty.map('P.TNotebook.Tab',background=[('selected',BG3)],foreground=[('selected',BRIGHT)])
        sty.configure('P.TFrame',background=BG)
        sty.configure('PA.Horizontal.TProgressbar',troughcolor=BG2,background=ACC,
                      thickness=3,borderwidth=0,relief='flat')

        nb=ttk.Notebook(self.root,style='P.TNotebook')
        nb.pack(fill=tk.BOTH,expand=True)

        t1=ttk.Frame(nb,style='P.TFrame'); nb.add(t1,text='  ▶  PLAY  ')
        t2=ttk.Frame(nb,style='P.TFrame'); nb.add(t2,text='  ⬇  EXPORT  ')
        t3=ttk.Frame(nb,style='P.TFrame'); nb.add(t3,text='  📄  SOURCE  ')

        self._build_play(t1)
        self._build_export(t2)
        self._build_source(t3)

        # ── bottom tray (always visible: volume + status)
        tk.Frame(self.root,bg=SEP,height=1).pack(fill=tk.X)
        tray=tk.Frame(self.root,bg=BG2,pady=5)
        tray.pack(fill=tk.X)

        tk.Label(tray,text='vol',font=('Courier',7),fg=MUTED,bg=BG2).pack(side=tk.LEFT,padx=(10,2))
        vol_sl=tk.Scale(tray,from_=0,to=100,orient='horizontal',
                        variable=tk.IntVar(),bg=BG2,fg=MUTED,
                        troughcolor=BG3,activebackground=ACC,
                        highlightthickness=0,bd=0,sliderlength=14,width=6,
                        showvalue=False,length=100,
                        command=self._vol_change)
        # bind DoubleVar → IntVar bridge
        self._vol_ivar=tk.IntVar(value=85)
        vol_sl.config(variable=self._vol_ivar)
        vol_sl.pack(side=tk.LEFT)
        self._vol_pct=tk.Label(tray,text='85%',font=('Courier',7),fg=MUTED,bg=BG2,width=4)
        self._vol_pct.pack(side=tk.LEFT,padx=(2,8))

        sep2=tk.Frame(tray,bg=SEP,width=1); sep2.pack(side=tk.LEFT,fill=tk.Y,pady=2)

        self._status_var=tk.StringVar(value='ready')
        tk.Label(tray,textvariable=self._status_var,font=('Courier',7),fg=MUTED,bg=BG2,
                 anchor='w').pack(side=tk.LEFT,padx=8)

        if self._no_player:
            tk.Label(tray,text='⚠ no audio',font=('Courier',7),fg='#886600',bg=BG2).pack(side=tk.RIGHT,padx=10)

    # ── PLAY tab ────────────────────────────────────────────────

    def _build_play(self,parent):
        inner=tk.Frame(parent,bg=BG,padx=16,pady=12)
        inner.pack(fill=tk.BOTH,expand=True)

        # AMBIENT
        self._amb_btn=tk.Button(inner,text='▶   AMBIENT',
            font=('Courier',15,'bold'),fg=TEAL,bg=BG2,
            activeforeground='#fff',activebackground='#16162e',
            relief='flat',bd=0,padx=44,pady=16,cursor='hand2',
            command=self._tog_ambient)
        self._amb_btn.pack(pady=(4,10))

        # progress
        self._prog_bar=ttk.Progressbar(inner,style='PA.Horizontal.TProgressbar',
            orient='horizontal',length=360,mode='determinate')
        self._prog_bar.pack(pady=(0,2))
        self._info=tk.Label(inner,text='',font=('Courier',7),fg=STAR,bg=BG)
        self._info.pack()

        # divider
        sep=tk.Frame(inner,bg=SEP,height=1); sep.pack(fill=tk.X,pady=10)
        self._row_label(inner,'🌿  NATURE SOUNDS')

        # nature grid
        grid=tk.Frame(inner,bg=BG); grid.pack(pady=(6,0))
        NATS=[('WIND','pure_wind'),('RAIN','pure_rain'),('OCEAN','pure_ocean'),
              ('WATER','pure_water'),('FIRE','pure_fire'),('STORM','pure_storm'),
              ('BOWLS','pure_bowls'),('CHIMES','pure_chimes'),('NATURE','pure_nature')]
        for idx,(label,mode) in enumerate(NATS):
            btn=tk.Button(grid,text=label,font=('Courier',8,'bold'),fg=NAT,bg=BG3,
                activeforeground='#fff',activebackground=NAT2,
                relief='flat',bd=0,padx=8,pady=7,width=7,cursor='hand2',
                command=lambda m=mode:self._tog_nature(m))
            btn.grid(row=idx//5,column=idx%5,padx=2,pady=2,sticky='nsew')
            self._nat_btns[mode]=btn

    # ── EXPORT tab ──────────────────────────────────────────────

    def _build_export(self,parent):
        # split: form left, options right
        left=tk.Frame(parent,bg=BG,padx=14,pady=10)
        left.pack(side=tk.LEFT,fill=tk.BOTH,expand=True)
        right=tk.Frame(parent,bg=PANEL,padx=12,pady=10,width=170)
        right.pack(side=tk.RIGHT,fill=tk.Y); right.pack_propagate(False)

        R=0
        def lbl(t,r,c=0,sticky='w'):
            tk.Label(left,text=t,font=('Courier',8),fg=MUTED,bg=BG).grid(row=r,column=c,sticky=sticky,pady=2,padx=(0,6))

        # mode
        lbl('mode',R)
        self._exp_mode=tk.StringVar(value='ambient')
        cb=ttk.Combobox(left,textvariable=self._exp_mode,values=ALL_MODES_LIST,width=18,state='readonly')
        cb.grid(row=R,column=1,sticky='w',pady=2); R+=1

        # duration
        lbl('duration (s)',R)
        self._exp_dur=tk.IntVar(value=180)
        ttk.Spinbox(left,from_=10,to=7200,textvariable=self._exp_dur,width=8).grid(row=R,column=1,sticky='w',pady=2); R+=1

        # quality
        lbl('quality',R)
        self._exp_qual=tk.StringVar(value='balanced')
        qf=tk.Frame(left,bg=BG); qf.grid(row=R,column=1,sticky='w',pady=2)
        for q in ('mobile','balanced','studio'):
            tk.Radiobutton(qf,text=q,variable=self._exp_qual,value=q,
                           font=('Courier',7),fg=MUTED,bg=BG,selectcolor=BG3,
                           activebackground=BG,activeforeground=BRIGHT).pack(side=tk.LEFT,padx=2)
        R+=1

        # bpm
        lbl('bpm (0=auto)',R)
        self._exp_bpm=tk.IntVar(value=0)
        ttk.Spinbox(left,from_=0,to=300,textvariable=self._exp_bpm,width=6).grid(row=R,column=1,sticky='w',pady=2); R+=1

        # root
        lbl('root note',R)
        self._exp_root=tk.StringVar(value='')
        tk.Entry(left,textvariable=self._exp_root,width=8,
                 bg=BG2,fg=BRIGHT,insertbackground=BRIGHT,relief='flat',bd=1
                 ).grid(row=R,column=1,sticky='w',pady=2); R+=1

        # seed
        lbl('seed (0=rand)',R)
        self._exp_seed=tk.IntVar(value=0)
        tk.Entry(left,textvariable=self._exp_seed,width=10,
                 bg=BG2,fg=BRIGHT,insertbackground=BRIGHT,relief='flat',bd=1
                 ).grid(row=R,column=1,sticky='w',pady=2); R+=1

        # output folder
        lbl('output folder',R)
        self._exp_out=tk.StringVar(value=str(Path.home()/'Documents'/'PyAmby'))
        ef=tk.Frame(left,bg=BG); ef.grid(row=R,column=1,sticky='w',pady=2)
        tk.Entry(ef,textvariable=self._exp_out,width=16,
                 bg=BG2,fg=BRIGHT,insertbackground=BRIGHT,relief='flat',bd=1).pack(side=tk.LEFT)
        tk.Button(ef,text='…',font=('Courier',8),fg=MUTED,bg=BG3,relief='flat',bd=0,
                  padx=4,command=self._browse).pack(side=tk.LEFT,padx=3)
        R+=1

        # generate button
        self._exp_btn=tk.Button(left,text='⬇  Generate WAV',
            font=('Courier',10,'bold'),fg=TEAL,bg=BG2,
            activeforeground='#fff',activebackground='#16162e',
            relief='flat',bd=0,pady=10,padx=20,cursor='hand2',
            command=self._exp_start)
        self._exp_btn.grid(row=R,column=0,columnspan=2,pady=(12,4)); R+=1

        # export progress
        self._exp_prog=ttk.Progressbar(left,style='PA.Horizontal.TProgressbar',
            orient='horizontal',length=280,mode='determinate')
        self._exp_prog.grid(row=R,column=0,columnspan=2,pady=2); R+=1
        self._exp_lbl=tk.Label(left,text='',font=('Courier',7),fg=MUTED,bg=BG)
        self._exp_lbl.grid(row=R,column=0,columnspan=2,pady=1); R+=1

        # open folder button (hidden until export done)
        self._exp_open_btn=tk.Button(left,text='open folder',
            font=('Courier',7),fg=ACC,bg=BG,relief='flat',bd=0,cursor='hand2',
            command=self._open_folder)
        # not packed yet — shown after success

        # right panel: flags
        tk.Label(right,text='FLAGS',font=('Courier',8,'bold'),fg=ACC2,bg=PANEL).pack(anchor='w',pady=(0,6))
        self._flag_vars={}
        for f in('vinyl','tape','wide','warm','cold','sparse','dense','bright','bowls','chimes'):
            v=tk.BooleanVar()
            tk.Checkbutton(right,text=f,variable=v,font=('Courier',7),
                           fg=MUTED,bg=PANEL,selectcolor=BG3,activebackground=PANEL,
                           activeforeground=BRIGHT).pack(anchor='w',pady=1)
            self._flag_vars[f]=v

    # ── SOURCE tab ──────────────────────────────────────────────

    def _build_source(self,parent):
        bar=tk.Frame(parent,bg=BG2,pady=3); bar.pack(fill=tk.X)
        tk.Label(bar,text=f'  embedded engine  ·  {len(CORE):,} chars  ·  read-only',
                 font=('Courier',7),fg=MUTED,bg=BG2).pack(side=tk.LEFT)
        tk.Frame(parent,bg=SEP,height=1).pack(fill=tk.X)
        txt=scrolledtext.ScrolledText(parent,wrap=tk.NONE,font=('Courier',7),
            fg='#90b890',bg='#060a06',insertbackground=ACC,relief='flat',bd=0)
        txt.pack(fill=tk.BOTH,expand=True)
        txt.insert(tk.END,CORE); txt.config(state=tk.DISABLED)

    # ── helpers ─────────────────────────────────────────────────

    def _row_label(self,parent,text):
        f=tk.Frame(parent,bg=BG); f.pack(fill=tk.X,pady=(0,2))
        tk.Label(f,text=text,font=('Courier',8,'bold'),fg=ACC2,bg=BG).pack(side=tk.LEFT)
        tk.Frame(f,bg=DIM,height=1).pack(side=tk.LEFT,fill=tk.X,expand=True,padx=(8,0),pady=6)

    def _deactivate(self):
        self._amb_btn.config(text='▶   AMBIENT',fg=TEAL,bg=BG2)
        for btn in self._nat_btns.values(): btn.config(fg=NAT,bg=BG3)
        self._mode=None
        self._prog_bar.stop(); self._prog_bar.configure(mode='determinate',value=0)

    def _vol_change(self,val):
        v=int(float(val)); pct=v/100.
        self._vol_var.set(pct); self.engine.set_vol(pct)
        self._vol_pct.config(text=f'{v}%')

    # ── play callbacks ──────────────────────────────────────────

    def _tog_ambient(self):
        if self._mode=='ambient':
            self.engine.stop(); self._deactivate(); self._set_status('stopped')
        else:
            self.engine.stop(); self._deactivate()
            self._amb_btn.config(text='◼   STOP',fg=RED,bg=BG2)
            self._mode='ambient'; self.engine.start(fixed=None)
            self._set_status('initializing …')

    def _tog_nature(self,mode):
        if self._mode==mode:
            self.engine.stop(); self._deactivate(); self._set_status('stopped')
        else:
            self.engine.stop(); self._deactivate()
            self._nat_btns[mode].config(fg='#ffffff',bg=NAT2)
            self._mode=mode; self.engine.start(fixed=mode)
            self._set_status(f"starting  {mode.replace('pure_','').replace('_',' ')} …")

    def _on_status(self,msg):
        self.root.after(0,lambda m=msg: self._set_status(m))
    def _on_prog(self,pct):
        def _u(p=pct):
            if self._mode: self._prog_bar.configure(mode='determinate',value=p)
        self.root.after(0,_u)
    def _on_play(self,_):
        def _u():
            if self._mode:
                self._prog_bar.configure(mode='indeterminate'); self._prog_bar.start(25)
        self.root.after(0,_u)
    def _set_status(self,msg):
        self._status_var.set(msg)
        if hasattr(self,'_info'): self._info.config(text=msg)

    # ── export callbacks ────────────────────────────────────────

    def _browse(self):
        d=filedialog.askdirectory()
        if d: self._exp_out.set(d)

    def _exp_start(self):
        if self._exp_rendering: return
        self._exp_rendering=True
        self._exp_last_path=None
        self._exp_btn.config(state=tk.DISABLED,text='⏳  rendering …')
        self._exp_prog['value']=0
        self._exp_lbl.config(text='')
        try: self._exp_open_btn.pack_forget()
        except: pass
        threading.Thread(target=self._exp_worker,daemon=True).start()

    def _exp_worker(self):
        try:
            # load engine if not yet loaded
            if not self.engine._loaded: self.engine._load()
            render=self.engine._ns['render']

            mode=self._exp_mode.get()
            dur=self._exp_dur.get()
            bpm=self._exp_bpm.get() or None
            root_s=self._exp_root.get().strip()
            root=note_to_midi(root_s) if root_s else None
            seed=self._exp_seed.get() or random.randint(0,2**32-1)
            flags={f for f,v in self._flag_vars.items() if v.get()}
            qual=self._exp_qual.get()
            out=self._exp_out.get()

            def prog(pct):
                self.root.after(0,lambda p=pct: self._exp_prog.configure(value=p))
                self.root.after(0,lambda p=pct: self._exp_lbl.config(text=f'rendering  {p}%'))

            params={'mode':mode,'duration':dur,'bpm':bpm,'root':root,'flags':flags,
                    'seed':seed,'quality':qual,'out_dir':out,'mp3':False,'volume':1.0}
            path=render(params,progress_cb=prog)
            self._exp_last_path=str(path)
            self.root.after(0,self._exp_done,str(path),None)
        except Exception as e:
            self.root.after(0,self._exp_done,None,str(e))

    def _exp_done(self,path,error):
        self._exp_rendering=False
        self._exp_btn.config(state=tk.NORMAL,text='⬇  Generate WAV')
        if error:
            self._exp_lbl.config(text=f'error: {error}',fg=RED)
        else:
            self._exp_lbl.config(text=f'saved  ·  {Path(path).name}',fg=TEAL)
            self._exp_open_btn.grid(row=10,column=0,columnspan=2,pady=3)

    def _open_folder(self):
        p=self._exp_last_path
        if not p: return
        folder=str(Path(p).parent)
        try:
            if sys.platform=='win32': os.startfile(folder)
            elif sys.platform=='darwin': subprocess.Popen(['open',folder])
            else: subprocess.Popen(['xdg-open',folder])
        except: pass

    # ── run ─────────────────────────────────────────────────────

    def run(self):
        self.root.protocol('WM_DELETE_WINDOW',self._quit)
        self.root.mainloop()
    def _quit(self):
        self.engine.stop()
        try: self.root.destroy()
        except: pass

if __name__=='__main__':
    PyAmby().run()
