# lazy-fitness
Lazy Fitness – AI-powered web app for personalized workouts and diet plans.
import React, { useState, useEffect, useCallback } from 'react';
import { User, UserProfile, FitnessGoal, WorkoutLocation, ExperienceLevel } from './types';
import { authService } from './services/authService';
import { generateWorkoutPlan, calculateDiet } from './constants';
import { getCoachMessage } from './services/geminiService';
import { Card, Button, Input, Select } from './components/UI';

const App: React.FC = () => {
  const [user, setUser] = useState<User | null>(authService.getCurrentUser());
  const [view, setView] = useState<'login' | 'signup' | 'onboarding' | 'dashboard'>(user ? (user.profile ? 'dashboard' : 'onboarding') : 'login');
  const [coachMsg, setCoachMsg] = useState<string>("");
  const [loadingCoach, setLoadingCoach] = useState(false);

  useEffect(() => {
    if (user && user.profile && view === 'dashboard') {
      loadCoach();
    }
  }, [user, view]);

  const loadCoach = async () => {
    if (!user) return;
    setLoadingCoach(true);
    const msg = await getCoachMessage(user);
    setCoachMsg(msg);
    setLoadingCoach(false);
  };

  const handleAuthSuccess = (userData: User) => {
    setUser(userData);
    if (userData.profile) {
      setView('dashboard');
    } else {
      setView('onboarding');
    }
  };

  const handleLogout = () => {
    authService.logout();
    setUser(null);
    setView('login');
  };

  if (!user && (view === 'login' || view === 'signup')) {
    return <AuthScreen view={view} setView={setView} onSuccess={handleAuthSuccess} />;
  }

  if (view === 'onboarding') {
    return <OnboardingScreen user={user!} onComplete={(updatedUser) => {
      setUser(updatedUser);
      setView('dashboard');
    }} />;
  }

  return (
    <div className="min-h-screen bg-zinc-950 pb-24">
      <header className="p-6 flex justify-between items-center border-b border-zinc-900 sticky top-0 bg-zinc-950 z-10">
        <h1 className="text-xl font-semibold tracking-tight text-emerald-500">ZenFit</h1>
        <button onClick={handleLogout} className="text-sm text-zinc-500 hover:text-zinc-300">Logout</button>
      </header>

      <main className="max-w-md mx-auto p-6 flex flex-col gap-8">
        {/* Coach Section */}
        <section>
          <div className="flex items-center gap-3 mb-4">
            <div className="w-10 h-10 rounded-full bg-emerald-500/10 flex items-center justify-center">
              <span className="text-emerald-500 text-lg">🌿</span>
            </div>
            <div>
              <h2 className="text-zinc-100 font-medium">Zen Coach</h2>
              <p className="text-xs text-zinc-500 uppercase tracking-widest">Mindful Guidance</p>
            </div>
          </div>
          <Card className="bg-emerald-500/5 border-emerald-500/10 italic text-zinc-300 leading-relaxed">
            {loadingCoach ? "Thinking..." : coachMsg || "Breathe in. Breathe out. You're exactly where you need to be."}
          </Card>
        </section>

        {/* Dashboard Stats */}
        <div className="grid grid-cols-2 gap-4">
          <Card className="p-4 flex flex-col items-center justify-center text-center">
            <span className="text-zinc-500 text-xs uppercase mb-1">Streak</span>
            <span className="text-2xl font-semibold text-zinc-100">{user?.streak || 0}</span>
            <span className="text-[10px] text-zinc-500">Days Active</span>
          </Card>
          <Card className="p-4 flex flex-col items-center justify-center text-center">
            <span className="text-zinc-500 text-xs uppercase mb-1">Weight</span>
            <span className="text-2xl font-semibold text-zinc-100">
              {user?.weightHistory.length ? user.weightHistory[user.weightHistory.length - 1].weight : '--'}
            </span>
            <span className="text-[10px] text-zinc-500">kg (Last check)</span>
          </Card>
        </div>

        {/* Today's Workout */}
        <section>
          <h3 className="text-lg font-medium mb-4 flex items-center gap-2">
            <span className="w-1 h-5 bg-emerald-500 rounded-full"></span>
            Today's Session
          </h3>
          <TodayWorkout user={user!} onWorkoutComplete={() => {
             if (!user) return;
             const today = new Date().toISOString().split('T')[0];
             if (user.lastWorkoutDate === today) return;
             
             const updatedUser = {
               ...user,
               streak: user.streak + 1,
               lastWorkoutDate: today
             };
             authService.updateUser(updatedUser);
             setUser(updatedUser);
          }} />
        </section>

        {/* Weekly Plan */}
        <section>
          <h3 className="text-lg font-medium mb-4 flex items-center gap-2">
            <span className="w-1 h-5 bg-emerald-500 rounded-full"></span>
            Weekly Path
          </h3>
          <WeeklyOverview user={user!} />
        </section>

        {/* Diet Section */}
        <section>
          <h3 className="text-lg font-medium mb-4 flex items-center gap-2">
            <span className="w-1 h-5 bg-emerald-500 rounded-full"></span>
            Nourishment
          </h3>
          <DietView user={user!} />
        </section>

        {/* Weight Tracker */}
        <section>
           <WeightTracker user={user!} onUpdate={(w) => {
             const updatedUser = {
               ...user!,
               weightHistory: [...user!.weightHistory, { date: new Date().toISOString(), weight: w }]
             };
             authService.updateUser(updatedUser);
             setUser(updatedUser);
           }} />
        </section>
      </main>
    </div>
  );
};

// --- Sub-Components ---

const AuthScreen: React.FC<{ view: 'login' | 'signup', setView: (v: 'login' | 'signup') => void, onSuccess: (u: User) => void }> = ({ view, setView, onSuccess }) => {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [name, setName] = useState("");
  const [error, setError] = useState("");

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    setError("");
    if (view === 'signup') {
      const user = authService.signup(email, password, name);
      if (user) onSuccess(user);
      else setError("User already exists");
    } else {
      const user = authService.login(email, password);
      if (user) onSuccess(user);
      else setError("Invalid credentials");
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center p-6 bg-zinc-950">
      <div className="w-full max-w-sm">
        <div className="mb-8 text-center">
          <div className="text-4xl mb-4 text-emerald-500">🌿</div>
          <h1 className="text-3xl font-semibold mb-2">ZenFit</h1>
          <p className="text-zinc-500">Mindful progress, minimum stress.</p>
        </div>
        <form onSubmit={handleSubmit} className="flex flex-col gap-4">
          {view === 'signup' && <Input label="Name" value={name} onChange={setName} required />}
          <Input label="Email" type="email" value={email} onChange={setEmail} required />
          <Input label="Password" type="password" value={password} onChange={setPassword} required />
          {error && <p className="text-red-400 text-xs text-center">{error}</p>}
          <Button type="submit" className="mt-2">{view === 'signup' ? 'Join ZenFit' : 'Login'}</Button>
          <Button variant="ghost" onClick={() => setView(view === 'signup' ? 'login' : 'signup')}>
            {view === 'signup' ? 'Already have an account? Login' : "Don't have an account? Sign up"}
          </Button>
        </form>
      </div>
    </div>
  );
};

const OnboardingScreen: React.FC<{ user: User, onComplete: (u: User) => void }> = ({ user, onComplete }) => {
  const [age, setAge] = useState("25");
  const [height, setHeight] = useState("175");
  const [weight, setWeight] = useState("70");
  const [goal, setGoal] = useState(FitnessGoal.STAY_HEALTHY);
  const [location, setLocation] = useState(WorkoutLocation.GYM);
  const [experience, setExperience] = useState(ExperienceLevel.BEGINNER);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    const profile: UserProfile = {
      age: parseInt(age),
      height: parseInt(height),
      weight: parseFloat(weight),
      goal,
      location,
      experience,
      onboarded: true
    };
    const updatedUser = { 
      ...user, 
      profile, 
      weightHistory: [{ date: new Date().toISOString(), weight: parseFloat(weight) }] 
    };
    authService.updateUser(updatedUser);
    onComplete(updatedUser);
  };

  return (
    <div className="min-h-screen p-6 bg-zinc-950">
      <div className="max-w-sm mx-auto pt-8">
        <h1 className="text-2xl font-semibold mb-2">Personalize your journey</h1>
        <p className="text-zinc-500 mb-8 text-sm">We'll create a low-pressure plan just for you.</p>
        <form onSubmit={handleSubmit} className="flex flex-col gap-5">
          <div className="flex gap-4">
             <Input label="Age" type="number" value={age} onChange={setAge} required />
             <Input label="Height (cm)" type="number" value={height} onChange={setHeight} required />
          </div>
          <Input label="Weight (kg)" type="number" value={weight} onChange={setWeight} required />
          <Select 
            label="Fitness Goal" 
            value={goal} 
            onChange={(v) => setGoal(v as FitnessGoal)}
            options={[
              { label: 'Stay Healthy', value: FitnessGoal.STAY_HEALTHY },
              { label: 'Muscle Gain', value: FitnessGoal.MUSCLE_GAIN },
              { label: 'Fat Loss', value: FitnessGoal.FAT_LOSS },
            ]}
          />
          <Select 
            label="Primary Location" 
            value={location} 
            onChange={(v) => setLocation(v as WorkoutLocation)}
            options={[
              { label: 'Gym', value: WorkoutLocation.GYM },
              { label: 'Home', value: WorkoutLocation.HOME },
            ]}
          />
          <Select 
            label="Experience" 
            value={experience} 
            onChange={(v) => setExperience(v as ExperienceLevel)}
            options={[
              { label: 'Beginner', value: ExperienceLevel.BEGINNER },
              { label: 'Intermediate', value: ExperienceLevel.INTERMEDIATE },
            ]}
          />
          <Button type="submit" className="mt-4">Build My Plan</Button>
        </form>
      </div>
    </div>
  );
};

const TodayWorkout: React.FC<{ user: User, onWorkoutComplete: () => void }> = ({ user, onWorkoutComplete }) => {
  const plan = generateWorkoutPlan(user.profile!);
  const days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'];
  const todayName = days[new Date().getDay()];
  const todayWorkout = plan.days.find(d => d.dayName === todayName);
  const [completed, setCompleted] = useState(user.lastWorkoutDate === new Date().toISOString().split('T')[0]);

  if (!todayWorkout) return <Card>Rest day. Enjoy the quiet.</Card>;

  if (todayWorkout.isRest) {
    return (
      <Card className="flex flex-col items-center gap-2 text-center border-dashed border-zinc-700">
        <span className="text-3xl">🧘</span>
        <h4 className="font-medium text-zinc-100">Rest Day</h4>
        <p className="text-sm text-zinc-500">Recovery is where the magic happens. Hydrate and relax.</p>
      </Card>
    );
  }

  return (
    <Card className={`relative overflow-hidden ${completed ? 'opacity-70' : ''}`}>
      <div className="flex justify-between items-start mb-6">
        <div>
          <h4 className="text-xl font-semibold text-zinc-100">{todayWorkout.type}</h4>
          <p className="text-sm text-zinc-500">{todayWorkout.exercises.length} Exercises</p>
        </div>
        {completed && <span className="text-emerald-500 font-medium text-sm flex items-center gap-1">✓ Done</span>}
      </div>

      <div className="space-y-4">
        {todayWorkout.exercises.map((ex, i) => (
          <div key={i} className="flex items-center gap-4 group">
            <div className="w-8 h-8 rounded-full bg-zinc-800 flex items-center justify-center text-xs font-bold text-zinc-500 group-hover:bg-emerald-500/20 group-hover:text-emerald-500 transition-colors">
              {i + 1}
            </div>
            <div className="flex-1">
              <p className="text-zinc-200 font-medium">{ex.name}</p>
              <p className="text-xs text-zinc-500">{ex.sets} sets · {ex.reps}</p>
            </div>
          </div>
        ))}
      </div>

      {!completed && (
        <Button onClick={() => { setCompleted(true); onWorkoutComplete(); }} className="w-full mt-6">
          Mark as Complete
        </Button>
      )}
    </Card>
  );
};

const WeeklyOverview: React.FC<{ user: User }> = ({ user }) => {
  const plan = generateWorkoutPlan(user.profile!);
  const days = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday'];
  const currentDay = days[new Date().getDay() - 1 === -1 ? 6 : new Date().getDay() - 1];

  return (
    <div className="space-y-3">
      {plan.days.map((day, i) => (
        <div 
          key={i} 
          className={`p-4 rounded-2xl border flex items-center justify-between transition-all ${
            day.dayName === currentDay 
            ? 'bg-emerald-500/10 border-emerald-500/30' 
            : 'bg-zinc-900 border-zinc-800'
          }`}
        >
          <div className="flex items-center gap-3">
            <div className={`w-2 h-2 rounded-full ${day.isRest ? 'bg-zinc-700' : 'bg-emerald-500'}`}></div>
            <div>
              <p className="text-sm font-medium text-zinc-100">{day.dayName}</p>
              <p className="text-[10px] uppercase text-zinc-500 tracking-wider">
                {day.isRest ? 'Rest & Recover' : day.type}
              </p>
            </div>
          </div>
          {!day.isRest && <span className="text-[10px] text-zinc-600 uppercase font-bold">{day.exercises.length} Ex</span>}
        </div>
      ))}
    </div>
  );
};

const DietView: React.FC<{ user: User }> = ({ user }) => {
  const diet = calculateDiet(user.profile!);

  return (
    <Card className="space-y-6">
      <div className="grid grid-cols-2 gap-4">
        <div className="bg-zinc-800/50 p-4 rounded-xl">
          <p className="text-[10px] text-zinc-500 uppercase mb-1">Target Calories</p>
          <p className="text-xl font-bold text-zinc-100">{diet.calories}</p>
        </div>
        <div className="bg-zinc-800/50 p-4 rounded-xl">
          <p className="text-[10px] text-zinc-500 uppercase mb-1">Daily Protein</p>
          <p className="text-xl font-bold text-zinc-100">{diet.protein}g</p>
        </div>
      </div>
      <div>
        <p className="text-sm font-medium mb-3 text-zinc-400">Gentle Meal Ideas</p>
        <ul className="space-y-2">
          {diet.meals.map((meal, i) => (
            <li key={i} className="flex gap-3 text-sm text-zinc-300">
              <span className="text-emerald-500">·</span>
              {meal}
            </li>
          ))}
        </ul>
      </div>
    </Card>
  );
};

const WeightTracker: React.FC<{ user: User, onUpdate: (w: number) => void }> = ({ user, onUpdate }) => {
  const [weight, setWeight] = useState("");
  const [showInput, setShowInput] = useState(false);

  return (
    <Card>
      <div className="flex justify-between items-center mb-4">
        <h4 className="font-medium">Progress Tracker</h4>
        {!showInput && <Button variant="ghost" className="text-xs h-8 px-2" onClick={() => setShowInput(true)}>Update</Button>}
      </div>
      
      {showInput ? (
        <div className="flex gap-2">
          <Input label="" type="number" placeholder="Weight (kg)" value={weight} onChange={setWeight} />
          <Button onClick={() => {
            if (weight) {
              onUpdate(parseFloat(weight));
              setWeight("");
              setShowInput(false);
            }
          }}>Save</Button>
          <Button variant="ghost" onClick={() => setShowInput(false)}>Cancel</Button>
        </div>
      ) : (
        <div className="h-24 flex items-end justify-between gap-1 mt-6">
           {user.weightHistory.slice(-10).map((h, i) => {
              const max = Math.max(...user.weightHistory.map(wh => wh.weight), 1);
              const min = Math.min(...user.weightHistory.map(wh => wh.weight), 0);
              const height = ((h.weight - min) / (max - min || 1)) * 60 + 20;
              return (
                <div key={i} className="flex-1 flex flex-col items-center gap-1">
                  <div 
                    className="w-full bg-emerald-500/20 rounded-t-sm transition-all duration-500" 
                    style={{ height: `${height}%` }}
                  ></div>
                  <span className="text-[8px] text-zinc-600">{new Date(h.date).toLocaleDateString([], { month: 'short', day: 'numeric'})}</span>
                </div>
              );
           })}
           {user.weightHistory.length === 0 && <p className="w-full text-center text-xs text-zinc-600 italic">No weight history yet</p>}
        </div>
      )}
    </Card>
  );
};

export default App;
