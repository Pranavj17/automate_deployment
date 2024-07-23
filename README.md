# automate_deployment
Automate Deployment pipeline for elixir umbrella application

**Implementation** Steps

1. Fetch the changes in the umbrella application from the main head
2. Get the changed apps path and set changed apps as ENV variable and call yout github repo
3. In the gitlab.yml file pass down the Global ENV to as local variable to your ci-generator.jsonnet
4. Use this condition to determine where to deploy always or manual after passing the necessary needs

**bootstrap.rb**
```
-- rest of code --

CHANGES = run_cmd('git diff --name-only origin/main...HEAD')
          .split("\n")
@changed_apps = []
def source_changed
  @changed_apps = CHANGES
                 .reject { |path| IGNORE.any? { |r| r =~ path } }
                 .map { |path| path.split('/')[1] }
                 .uniq
                 .select { |c| ALL.member?(c) }

  ALL.select { |app| dependents(app).any? { |dependency| @changed_apps.include?(dependency) } }
end

changed_projects = if ENV['CI_COMMIT_REF_NAME'] == 'main'
  ALL # for main, build and test all apps irrespective of changes
else
  source_changed.uniq
end

if changed_projects.empty?
  puts 'No project changes detected'
  exit(true)
end

puts "-----changed_apps-----"
puts @changed_apps.to_s


variables = {
  'CHANGED_APPS' => @changed_apps.to_s
}

changed_projects.each do |project|
  variables["BUILD_#{project.upcase}"] = true
end

command = 'curl -s --request POST'
command += variables.map { |key, value| " --form 'variables[#{key}]=#{value}'" }.join(' ')
command += '** CI PROJECT PIPELINE LINK'
puts command
exit(system(command))

```
**gitlab-ci.yml**
```
for project in ./apps/*; do
  jsonnet ci-generator.jsonnet --ext-str changed_apps="$CHANGED_APPS" --ext-str apps=$(basename $project) --ext-str settings="$(cat .gitlab-ci.yml | yq '.[".settings"]')" > .gitlab/ci/$(basename $project)-config.yml
done
```

**ci-generator.jsonnet**
```
{
  ['immortal']: {
    # -- rest of code --
    when: if std.extVar('ref_name') == "immortal" && std.member(std.extVar('changed_apps'), app_name) then 'always' else 'manual',
  },
  for app_name in std.split(std.extVar('apps'), '\n')
  if src.settings[app_name].deploy
}
```
