# CodeGet-Wiki

I am looking for suggestions - How can I use shared projects in multiple dotnet solutions?                            
● Few solid approaches:                                                                                                 
  1. NuGet Packages (best for stable, versioned libs)                                                                   
  Pack shared project → publish to private feed (Azure Artifacts, GitHub Packages, local folder feed). Consume via        dotnet add package. Best when API is stable and you want explicit versioning.

  2. Git Submodules

  Add shared repo as submodule inside each solution repo. Reference project directly via <ProjectReference>. Keeps
  source editable across solutions but submodule management is painful.

  3. Directory.Build.props + Relative Paths

  Place shared projects in a common folder adjacent to solution folders:
  ```bash
/repos
    /shared
      MyShared.csproj
    /SolutionA
      SolutionA.sln
    /SolutionB
      SolutionB.sln
```
  Reference via relative path in each .csproj:
  <ProjectReference Include="..\..\shared\MyShared.csproj" />
  Simple, no tooling. Breaks if repo layouts diverge.

  4. Central Package Management + Local NuGet Feed

  Use Directory.Packages.props for version pinning. Point NuGet config at local folder feed for inner-loop dev, swap to
  real feed in CI.

  5. Mono-repo (nuclear option)

  All solutions live in one repo. Single Directory.Build.props governs everything. ProjectReferences just work. Big
  upfront cost, simplest long-term.

  ---
  Recommendation: If shared code changes frequently alongside consumers → relative path refs or mono-repo. If shared
  code is a stable utility layer → NuGet package with private feed. Git submodules work but cause the most friction over
   time.

  What's your current repo structure? Can narrow the recommendation.

✻ Crunched for 11s

> Can I use environment variables, instead of relative paths to include external projects?                              
● Yes. MSBuild evaluates environment variables as properties.

```xml
  <ProjectReference Include="$(SHARED_PROJECTS_ROOT)\MyShared\MyShared.csproj" />
```

Set SHARED_PROJECTS_ROOT in your environment before building. Works in VS, dotnet build, CI pipelines.

Better pattern — define fallback in Directory.Build.props so devs don't need env var set if they use standard layout:

```xml
  <!-- Directory.Build.props -->
  <PropertyGroup>
    <SharedRoot Condition="'$(SHARED_PROJECTS_ROOT)' != ''">$(SHARED_PROJECTS_ROOT)</SharedRoot>
    <SharedRoot Condition="'$(SharedRoot)' == ''">$(MSBuildThisFileDirectory)..\shared</SharedRoot>
  </PropertyGroup>
```

  Then in .csproj:
```xml
  <ProjectReference Include="$(SharedRoot)\MyShared\MyShared.csproj" />
```

  CI: set SHARED_PROJECTS_ROOT in pipeline env. Dev: set in machine env or .env loaded by your shell. Falls back to
  relative path if unset.

  One caveat: Visual Studio resolves these at load time — if env var missing when VS opens, solution load fails.
  Fallback default above prevents that.
