Where the Speed Comes From

    1. Storage: 128 bits vs ~16KB per agent
    A locked winner is stored as 4 integers (~16 bytes). The equivalent hardcoded system prompt would be 
    ~4,000 tokens (~16,000 bytes). That's a 1,000x reduction in storage per agent.

    2. Selection: Numeric scoring vs text comparison
    Selecting from 32 candidates means scoring capabilityField() vectors (5 floats each) against a task  
    context — pure arithmetic. The old ART system compared 25-dimensional cosine similarity vectors      
    against stored centroids. ZeroSpecialist's selection is O(1) per candidate with no string operations.

    3. API payload: Minimal prompt overhead
    The generated system prompt is lean by design — it's algorithmically constructed from the capability 
    profile dimensions (verbosity, assertiveness, formality, domain focus, reasoning depth), not a       
    hand-authored essay. This means fewer tokens sent to the LLM per request, which directly reduces     
    latency.

    4. No database reads during coordination
    The candidate pool lives in Zustand (in-memory). The old system hit IndexedDB (Dexie) to load        
    SpecialistProfile records with their full execution histories and task signature vectors.
    ZeroSpecialist's coordination path is entirely in-memory until execution recording.

    The Addressable Space

    The seed space is 65,536 x 65,536 = 4,294,967,296 (4.3 billion) unique agent coordinates. Every one  
    of those agents is instantly reconstructable from its seed — no pre-generation, no storage. The      
    entire roster of 4.3 billion potential specialists "exists" as a mathematical field, and the swarm   
    samples from it like a procedural world generator.

    By comparison, a hardcoded system storing 4.3 billion system prompts at 4,000 tokens each would      
    require ~17 terabytes.

    Why Sonnet 4.6 Responses Feel Instant

    The coordination overhead — selecting a specialist, generating its prompt, matching an approach,     
    reading signals — now completes in single-digit milliseconds. The bottleneck shifts entirely to the  
    LLM inference itself. With Sonnet 4.6 (which is already fast) and web search enabled for grounding,  
    the total round-trip stays near-instant because ZeroSpecialist adds effectively zero latency to the  
    pipeline. The swarm coordination that used to involve multiple IndexedDB reads, vector math on stored
     profiles, and ART vigilance calculations is now a handful of integer hashes and float comparisons in
     memory.