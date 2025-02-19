# Solution

for the challenge found at: https://ctf.hackin.ca/challenges#spag-160fcsc2022-forensics-a-l-ancienne/

with the hint: "J'aime les spaghetti avec du fromage, beaucoup de fromage. Et beaucoup de spaghetti.
java -jar Fromage.jar 'JFFI{MonFlag}'"
---
## Here is my solution:

### Analysis:
I downloaded the the jar file, and opened it first using jadx to see its content quickly.
I immediately noticed that there are more than 20 similar files that require searching, so I decided to decompile it online, and open it with AndroidStudio.

### Code Understanding
In here, I spent some time understanding it. 
This code starts here:
```java
public static void main(String[] var0) {
      if (var0.length == 1 && var0[0].length() == 24 && Comté.miam(var0[0], 0)) {
         System.out.println("Bon flag: " + var0[0]);
      } else {
         System.out.println("Mauvais flag");
      }

   }
```
this `Comte.miam` is a function in a different file. it is being called to check if the provided argument (flag) is the correct flag or not.

I also searched for the `True` case that makes the provided flag correct to find it in `Roquefort.java` class.

I immediately realised that it is impossible to bruteforce for the values, and that doing it manually will not work either. with the help of ChatGPT and Claud, I was able to contruct scripts that facilitate automating data extraction and path solver

### data extraction
I used this script to extract transitions from java files:
```python
import os
import re

def parse_java_file(file_path):
    """Parse a single Java file to extract transitions"""
    with open(file_path, 'r', encoding='utf-8') as f:
        content = f.read()
    
    # Get class name
    class_match = re.search(r'public class (\w+)', content)
    if not class_match:
        return None, {}
    
    class_name = class_match.group(1)
    transitions = {}
    
    # Extract all case transitions
    pattern = r"case '(.)':\s*return (\w+)\.miam\("
    for match in re.finditer(pattern, content):
        char, target_class = match.groups()
        transitions[char] = target_class
    
    # Handle self-references (return miam)
    self_refs = re.finditer(r"case '(.)':\s*return miam\(", content)
    for match in self_refs:
        char = match.group(1)
        transitions[char] = class_name
        
    return class_name, transitions

def process_directory(directory):
    """Process all Java files in directory"""
    all_transitions = {}
    
    for filename in os.listdir(directory):
        if filename.endswith('.java'):
            file_path = os.path.join(directory, filename)
            class_name, transitions = parse_java_file(file_path)
            if class_name:
                all_transitions[class_name] = transitions
    
    # Save to file
    with open('transitions.py', 'w', encoding='utf-8') as f:
        f.write('transitions = {\n')
        for class_name, trans in sorted(all_transitions.items()):
            f.write(f"    '{class_name}': {{\n")
            for char, target in sorted(trans.items()):
                f.write(f"        '{char}': '{target}',\n")
            f.write('    },\n')
        f.write('}\n')

if __name__ == "__main__":
    directory = input("Enter the directory path containing the Java files: ")
    process_directory(directory)
    print("Transitions have been saved to 'transitions.py'")
```

this script generated a python file called transitions.py with all the transitions as follows:
```python
transitions = {
    'Abondance': {
        'A': 'Reblochon',
        'B': 'MontDOr',
        'C': 'SaintMarcellin',
        'D': 'Cantal',
        'E': 'Abondance',
        'F': 'Langres',
        'G': 'Chabichou',
        'H': 'Banon',
        'I': 'Camembert',
        'J': 'Langres',
        'K': 'Morbier',
        'L': 'Raclette',
        'M': 'Comté',
        'N': 'Époisses',
        'O': 'CapriceDesDieux',
        'P': 'BrieDeMelun',
        'Q': 'CapriceDesDieux',
        'R': 'Beaufort',
        'S': 'BrieDeProvins',
        'T': 'CapriceDesDieux',
        'U': 'Banon',
        'V': 'PoulignySaintPierre',
        'W': 'Camembert',
        'X': 'Cantal',
        'Y': 'Cantal',
        'Z': 'BrieDeProvins',
        '{': 'Époisses',
        '}': 'Banon',
    },
    'Angelot': {
        'A': 'Langres',
        'B': 'Cantal',
        'C': 'PontLÉvêque',
        'D': 'Charolais',
        'E': 'Époisses',
        'F': 'Camembert',
        'G': 'Charolais',
        'H': 'Chavroux',
        'I': 'Bougon',
        'J': 'Charolais',
        'K': 'Époisses',
        'L': 'Barousse',
        'M': 'SaintFélicien',
        'N': 'Emmental',
        'O': 'Autun',
        'P': 'Beaufort',
        'Q': 'Caillebotte',
        'R': 'Barousse',
        'S': 'Beaumont',
        'T': 'BrieDeMeaux',
        'U': 'BrieDeNangis',
        'V': 'Emmental',
        .
        .
        .
```

### correct path finding

From that point, the challenge is translated into a leetcode question to find the path from Comté to Roquefort with 24 character flag. With some testing and the help of Claud.ai, I made this script:

```python
import queue
from transitions import transitions  # Import the transitions we extracted

def find_flag(start_class='Comté', target_class='Roquefort', length=24):
    """Find a valid flag starting from start_class and ending at target_class"""
    # Initialize the search
    q = queue.Queue()
    q.put((start_class, 0, []))  # (current_class, position, path)
    visited = set()
    
    # Breadth-first search through transitions
    while not q.empty():
        current_class, pos, path = q.get()
        
        # Check if we've found a valid path
        if pos == length:
            if current_class == target_class:
                return ''.join(path)
            continue
            
        # Skip if we've seen this state
        state = (current_class, pos)
        if state in visited:
            continue
        visited.add(state)
        
        # Try each possible transition
        if current_class in transitions:
            for char, next_class in transitions[current_class].items():
                q.put((next_class, pos + 1, path + [char]))
    
    return None

def main():
    print("Searching for flag...")
    flag = find_flag()
    
    if flag:
        print(f"\nFound flag: {flag}")
    else:
        print("\nNo valid flag found")

if __name__ == "__main__":
    main()

```
which used Breadth-First Search to find the flag

its output is:
Searching for flag...

Found flag: JFFI{LEFROMAGECESTLAVIE}

which concludes this solution