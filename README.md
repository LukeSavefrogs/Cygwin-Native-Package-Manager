# Cygwin Package Manager (native)
## Background 
There are many alternatives out there for having an **apt-like** Package Manager for Cygwin (as well documented [here](https://stackoverflow.com/questions/9260014/how-do-i-install-cygwin-components-from-the-command-line/23143997#23143997)) like:
- [**apt-cyg**](https://github.com/transcode-open/apt-cyg)
- [**cyg-apt**](https://github.com/nylen/cyg-apt)
- The **GUI** (too mainstream)
- Some more...

But, as a challange, i **didn't** want to **install anything** and do everything i needed using **only** Cygwin inbuilt CLI commands. 

On Cygwin FAQ webpage at chapter **2.3** you can find a [list of all the parameters](https://cygwin.com/faq.html) you can pass to the setup to automate things, like specify which architecture, proxy to use AND which packages to **install** or **uninstall**.
So i just created a **wrapper** around these parameters. 

## Instructions
To use it:
- Copy this code into your `~/.bashrc`:
```Bash
CustomPkgManager () (
        local default='\033[0m';
        local red='\033[31m';
        local yellow='\033[33m'
        local cyan='\033[36m';
        local underlined='\033[4m';


        local operative_system=$(uname -o);
        local arch=$(uname -m);

        local installer_path=/tmp/CygwinSetup_${arch}.exe;
        local max_width_column=55;

        local terminal_lines=$(tput lines);
        local terminal_columns=$(tput cols);


        main () {
                local option=${1,,};
                shift;
                
                # This only works with Cygwin, so exclude others
                if [[ $operative_system != "Cygwin" ]]; then
                        printf "[${red}ERROR${default}] - Operative System is not Cygwin!\n\n";
                        return 1;
                fi

                # Check if there are any options
                [[ -z "$option" ]] && {
                        printf "[${red}ERROR${default}] - First paramtere not specified\n\n";

                        usage;
                        return 1;
                }

                # If there is no installer at the specified path, download it
                if [[ ! -f "$installer_path" ]]; then
                        printf "[${yellow}WARNING${default}] - Installer not found under path $installer_path\n";

                        download_setup;
                fi;
                
                # Do things based on the option specified as first parameter
                case $option in
                        "list")
                                printf "Downloading packages from repository. Please wait...";
                                list_of_packages=$(packages_from_repo);

                                # Clear line
                                printf "\r%*s" $terminal_columns "";

                                less <<< "$list_of_packages";


                                return 0;
                                ;;
                        "search")
                                search_params=$(echo "$@" | tr -s '[:space:]' | tr ',' '|' | tr ' ' '|');
                                [[ -z "$search_params" ]] && {
                                        printf "[${red}ERROR${default}] - No search params specified\n\n";

                                        usage;
                                        return 1;
                                }


                                printf "Downloading packages from repository. Please wait...";
                                formatted_data=$(packages_from_repo | awk "\$1 ~ /$search_params/{print}" | sed -r "s/$search_params/\\\\033[1;31m&\\\\033[0m/");

                                # Check if there are results
                                total_results=0;
                                [[ ! -z "$formatted_data" ]] && total_results=$(wc -l <<<"$formatted_data");

                                # Clear line before printing data
                                printf "\r%*s\r" $terminal_columns "";


                                # Print data
                                printf "\nTotal results: $total_results\n\n"
                                [[ ! -z "$formatted_data" ]] && printf "$formatted_data\n";


                                return 0;
                                ;;
                        "install")
                                packages=$(echo "$@" | tr -s '[:space:]' | tr ' ' ',');
                                [[ -z "$packages" ]] && {
                                        printf "[${red}ERROR${default}] - No packages specified\n\n";

                                        usage;
                                        return 1;
                                }


                                printf "Installing: $packages\n\n";
                                "$installer_path" -nq -WB -P $packages


                                return 0;
                                ;;
                        "remove")
                                packages=$(echo "$@" | tr -s '[:space:]' | tr ' ' ',');
                                [[ -z "$packages" ]] && {
                                        printf "[${red}ERROR${default}] - No packages specified\n\n";

                                        usage;
                                        return 1;
                                }


                                printf "Removing: $packages\n\n";
                                "$installer_path" -nq -WB -x $packages


                                return 0;
                                ;;
                        *)
                                printf "ERROR - Unknown option\n\n";
                                usage;

                                return 1;
                                ;;
                esac;
        }

        # ---------------- Functions ----------------
        usage() {
                printf "A small Package Manager for Cygwin made using only Cygwin inbuilt features\n\n";
                printf "Usage:\n";
                printf "\t${FUNCNAME[2]} list\n";
                printf "\t${FUNCNAME[2]} search ${underlined}REGEX${default}\n";
                printf "\t${FUNCNAME[2]} install ${underlined}PACKAGE${default}\n";
                printf "\t${FUNCNAME[2]} remove ${underlined}PACKAGE${default}\n";
        }


        download_setup() {
                printf "Download: ";
                [[ $arch == "x86_64" ]] && {
                        download_url='https://cygwin.com/setup-x86_64.exe';
                } || {
                        download_url='https://cygwin.com/setup-x86.exe';
                }


                curl -s "$download_url" > "$installer_path" && printf "${green}OK${default}\n\n" || printf "${red}ERRORE${default}\n\n";
                chmod 777 "$installer_path";
        }


        # Retrieves packages and descriptions from the package list hosted on Cygwin Official Website
        packages_from_repo() (
                data=$(curl -s --ssl-no-revoke https://cygwin.com/packages/package_list.html);

                awk -v max_length=$max_width_column '
                        /href="summary/{
                                line=$0;
                                name="";
                                description="";

                                name=substr(line, index(line, ".html\">")+7);
                                name=substr(name, 0, index(name, "</a>")-1);

                                description=substr(line, index(line, "</a></td><td>")+13);
                                description=substr(description, 0, index(description, "</td></tr>")-1);

                                printf("%-*s - %s\n", max_length, name, description);
                        }
                ' <<<"$data";
        );

        main "$@";

        return $?;
)
```
Then, to use it you can call the function with apt-like parameters like this:
- `CustomPkgManager install sl`
- `CustomPkgManager search nano`

## Tips 
To make everything more user-friendly you can use handy and easy-to-remember aliases: 
```Bash
alias pkgInstall='CustomPkgManager install'
alias pkgRemove='CustomPkgManager remove'
alias pkgList='CustomPkgManager list'
alias pkgSearch='CustomPkgManager search'
```
