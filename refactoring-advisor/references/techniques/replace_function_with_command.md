# Replace Function with Command

## Motivation

Encapsulates a function into a command object, enabling undo, builder-style parameters, inheritance hooks, and decomposition into sub-methods that can be tested independently. Only apply when you specifically need these facilities -- prefer plain functions ~95% of the time.

## Mechanics

- Create an empty class for the function. Name it based on the function.
- Use Move Function to move the function to the empty class.

  Keep the original function as a forwarding function until at least the end of the refactoring.

  Follow any convention the language has for naming commands. If there is no convention, choose a generic name for the command's execute function, such as "execute" or "call".

- Consider making a field for each argument, and move these arguments to the constructor.

## Example

Here is a function that scores points for an insurance application:

```
function score(candidate, medicalExam, scoringGuide) {
  let result = 0;
  let healthLevel = 0;
  let highMedicalRiskFlag = false;

  if (medicalExam.isSmoker) {
    healthLevel += 10;
    highMedicalRiskFlag = true;
  }
  let certificationGrade = "regular";
  if (scoringGuide.stateWithLowCertification(candidate.originState)) {
    certificationGrade = "low";
    result -= 5;
  }
  // lots more code like this
  result -= Math.max(healthLevel - 5, 0);
  return result;
}
```

I begin by creating an empty class and then using Move Function to move the function into it.

```
function score(candidate, medicalExam, scoringGuide) {
  return new Scorer().execute(candidate, medicalExam, scoringGuide);
}
```

class Scorer...

```
execute (candidate, medicalExam, scoringGuide) {
  let result = 0;
  let healthLevel = 0;
  let highMedicalRiskFlag = false;

  if (medicalExam.isSmoker) {
    healthLevel += 10;
    highMedicalRiskFlag = true;
  }
  let certificationGrade = "regular";
  if (scoringGuide.stateWithLowCertification(candidate.originState)) {
    certificationGrade = "low";
    result -= 5;
  }
  // lots more code like this
  result -= Math.max(healthLevel - 5, 0);
  return result;
}
```

I prefer to pass arguments to a command on the constructor and have the `execute` method take no parameters. This is very handy when I want to manipulate the command with a more complicated parameter setting lifecycle or customizations. Different command classes can have different parameters but be mixed together when queued for execution.

I can do these parameters one at a time.

```
function score(candidate, medicalExam, scoringGuide) {
  return new Scorer(candidate).execute(candidate, medicalExam, scoringGuide);
}
```

class Scorer...

```
constructor(candidate){
  this._candidate = candidate;
}

execute (candidate, medicalExam, scoringGuide) {
  let result = 0;
  let healthLevel = 0;
  let highMedicalRiskFlag = false;

  if (medicalExam.isSmoker) {
    healthLevel += 10;
    highMedicalRiskFlag = true;
  }
  let certificationGrade = "regular";
  if (scoringGuide.stateWithLowCertification(this._candidate.originState)) {
    certificationGrade = "low";
    result -= 5;
  }
  // lots more code like this
  result -= Math.max(healthLevel - 5, 0);
  return result;
}
```

I continue with the other parameters.

```
function score(candidate, medicalExam, scoringGuide) {
  return new Scorer(candidate, medicalExam, scoringGuide).execute();
}
```

class Scorer...

```
constructor(candidate, medicalExam, scoringGuide){
  this._candidate = candidate;
  this._medicalExam = medicalExam;
  this._scoringGuide = scoringGuide;
}


execute () {
  let result = 0;
  let healthLevel = 0;
  let highMedicalRiskFlag = false;

  if (this._medicalExam.isSmoker) {
    healthLevel += 10;
    highMedicalRiskFlag = true;
  }
  this._certificationGrade = "regular";
  if (this._scoringGuide.stateWithLowCertification(this._candidate.originState)) {
    this._certificationGrade = "low";
    result -= 5;
  }
  // lots more code like this
  result -= Math.max(healthLevel - 5, 0);
  return result;
}
```

That completes Replace Function with Command, but the whole point is to allow me to break down complicated functions. My next move is to change all the local variables into fields, one at a time.

class Scorer...

```
execute () {
  this._result = 0;
  this._healthLevel = 0;
  this._highMedicalRiskFlag = false;

  if (this._medicalExam.isSmoker) {
    this._healthLevel += 10;
    this._highMedicalRiskFlag = true;
  }
  this._certificationGrade = "regular";
  if (this._scoringGuide.stateWithLowCertification(this._candidate.originState)) {
    this._certificationGrade = "low";
    this._result -= 5;
  }
  // lots more code like this
  this._result -= Math.max(this._healthLevel - 5, 0);
  return this._result;
}
```

Now I've moved all the function's state to the command object, I can use refactorings like Extract Function without getting tangled up in all the variables and their scopes.

class Scorer...

```
execute () {
  this._result = 0;
  this._healthLevel = 0;
  this._highMedicalRiskFlag = false;

  this.scoreSmoking();
  this._certificationGrade = "regular";
  if (this._scoringGuide.stateWithLowCertification(this._candidate.originState)) {
    this._certificationGrade = "low";
    this._result -= 5;
  }
  // lots more code like this
  this._result -= Math.max(this._healthLevel - 5, 0);
  return this._result;
}

scoreSmoking() {
  if (this._medicalExam.isSmoker) {
    this._healthLevel += 10;
    this._highMedicalRiskFlag = true;
  }
}
```

This allows me to treat the command similarly to how I'd deal with a nested function.
