Le kit est créé. Voici la structure :                   
                                                                                          
  kit/
  ├── spec.yaml                    # mixin : autorise les ports host.docker.internal                                                                        
  └── files/                                                                                                                                                
      └── home/                                                                                                                                             
          └── .claude.json         # pré-configure le serveur MCP dans Claude Code                                                                          
                                                                                                                                                            
  ---
  Commandes pour démarrer une nouvelle sandbox                                                                                                              
                                                                                                                                                            
  1. Sur la machine hôte — autoriser le port de la gateway
                                                                                                                                                            
  sbx policy allow network localhost:9011                         
                                                                                                                                                            
  2. Sur la machine hôte — démarrer le stack MCP                  

  docker compose -f mcp-scripts/compose.dev.yml up -d                                                                                                       
  
  3. Sur la machine hôte — créer la sandbox avec le kit                                                                                                     
                                                                  
  sbx create claude --kit ./kit <chemin-workspace>
                                                                                                                                                            
  Exemple avec le répertoire courant :
                                                                                                                                                  ---
  sbx create claude --kit ./kit .   
  sbx run claude-sbx-calls-mcp-server                              
---                                                                                                                                                        
  Optionnel — injecter le kit dans une sandbox déjà existante
                                                                                                                                                            
  sbx kit add <nom-sandbox> ./kit                                 
                                                                                                                                                            
  ---
  Ce que fait le kit :                                                                                                                                      
  - spec.yaml : ajoute host.docker.internal:9011 et :8080 à la policy réseau de la sandbox (sans ce mixin, les connexions vers la gateway sont bloquées par
  défaut)                                                                                                                                                  
  - files/home/.claude.json : pré-configure le serveur MCP mcp-scripts dans Claude Code, qui pointe vers http://host.docker.internal:9011/mcp — la sandbox  
  est prête dès le démarrage, sans configuration manuelle
                                                            