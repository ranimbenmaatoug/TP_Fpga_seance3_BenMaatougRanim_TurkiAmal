# TP_Fpga_seance3_BenMaatougRanim_TurkiAmal_2G2_TD3_TP5

# Visualisation de la sortie HDMI
 La sortie HDMI de la carte est visualisée sur le PC grâce à un adaptateur HDMI-USB, reconnu comme un périphérique de capture vidéo, et au logiciel VLC (Média > Ouvrir un périphérique de capture).
 <img width="675" height="691" alt="visualisationHDMI" src="https://github.com/user-attachments/assets/0a0ba706-5145-4d83-95e9-23a2417368e3" />
 
# Contrôleur HDMI
## Code VHDL – hdmi_controler.vhd
```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity hdmi_controler is
    generic (
        h_res  : positive := 720;
        v_res  : positive := 480;
        h_sync : positive := 61;
        h_fp   : positive := 58;
        h_bp   : positive := 18;
        v_sync : positive := 5;
        v_fp   : positive := 30;
        v_bp   : positive := 9
    );
    port (
        i_clk   : in  std_logic;
        i_rst_n : in  std_logic;

        o_hdmi_hs : out std_logic;
        o_hdmi_vs : out std_logic;
        o_hdmi_de : out std_logic;

        o_pixel_en      : out std_logic;
        o_pixel_address : out natural;
        o_x_counter     : out natural;
        o_y_counter     : out natural
    );
end entity hdmi_controler;

architecture rtl of hdmi_controler is
    constant h_start : positive := h_sync + h_fp;   
    constant h_end   : positive := h_start + h_res;  
    constant h_total : positive := h_end + h_bp;     

    constant v_start : positive := v_sync + v_fp;
    constant v_end   : positive := v_start + v_res;
    constant v_total : positive := v_end + v_bp;

    signal r_h_count  : natural range 0 to h_total := 0;
    signal r_h_active : std_logic := '0';
    signal r_v_count  : natural range 0 to v_total := 0;
    signal r_v_active : std_logic := '0';
    signal s_x, s_y   : natural := 0;
begin

    
    process(i_clk, i_rst_n)
    begin
        if (i_rst_n = '0') then
            r_h_count  <= 0;
            o_hdmi_hs  <= '1';
            r_h_active <= '0';
        elsif rising_edge(i_clk) then
            -- Compteur de 0 à h_total
            if (r_h_count = h_total) then
                r_h_count <= 0;
            else
                r_h_count <= r_h_count + 1;
            end if;

            if (r_h_count >= h_sync) and (r_h_count /= h_total) then
                o_hdmi_hs <= '1';
            else
                o_hdmi_hs <= '0';
            end if;

            if (r_h_count = h_start) then
                r_h_active <= '1';
            elsif (r_h_count = h_end) then
                r_h_active <= '0';
            end if;
        end if;
    end process;

    
    process(i_clk, i_rst_n)
    begin
        if (i_rst_n = '0') then
            r_v_count  <= 0;
            o_hdmi_vs  <= '1';
            r_v_active <= '0';
        elsif rising_edge(i_clk) then
            if (r_h_count = h_total) then
                -- Compteur de 0 à v_total
                if (r_v_count = v_total) then
                    r_v_count <= 0;
                else
                    r_v_count <= r_v_count + 1;
                end if;

                if (r_v_count >= v_sync) and (r_v_count /= v_total) then
                    o_hdmi_vs <= '1';
                else
                    o_hdmi_vs <= '0';
                end if;

                if (r_v_count = v_start) then
                    r_v_active <= '1';
                elsif (r_v_count = v_end) then
                    r_v_active <= '0';
                end if;
            end if;
        end if;
    end process;

   
    process(i_clk, i_rst_n)
    begin
        if (i_rst_n = '0') then
            o_hdmi_de <= '0';
        elsif rising_edge(i_clk) then
            o_hdmi_de <= r_v_active and r_h_active;
        end if;
    end process;

   
    o_pixel_en <= r_v_active and r_h_active;

   
    s_x <= r_h_count - (h_start + 1) when (r_h_active = '1') else 0;
    s_y <= r_v_count - (v_start + 1) when (r_v_active = '1') else 0;

    o_x_counter <= s_x;
    o_y_counter <= s_y;

    o_pixel_address <= s_y * h_res + s_x;

end architecture rtl; 
```

## Code VHDL – tb_hdmi_controler.vhd
```  vhdllibrary ieee;
use ieee.std_logic_1164.all;

entity tb_hdmi_controler is
end entity tb_hdmi_controler;

architecture sim of tb_hdmi_controler is
    constant C_PERIOD : time := 37 ns;

    signal s_clk   : std_logic := '0';
    signal s_rst_n : std_logic := '0';
    signal s_hs, s_vs, s_de, s_pixel_en : std_logic;
    signal s_addr, s_x, s_y : natural;
    signal s_end   : boolean := false;
begin

    dut : entity work.hdmi_controler
        port map (
            i_clk           => s_clk,
            i_rst_n         => s_rst_n,
            o_hdmi_hs       => s_hs,
            o_hdmi_vs       => s_vs,
            o_hdmi_de       => s_de,
            o_pixel_en      => s_pixel_en,
            o_pixel_address => s_addr,
            o_x_counter     => s_x,
            o_y_counter     => s_y
        );

    s_clk <= not s_clk after C_PERIOD/2 when not s_end else '0';

    process
    begin
        s_rst_n <= '0';
        wait for 5*C_PERIOD;
        s_rst_n <= '1';
        wait for 3 * 858 * 525 * C_PERIOD;
        s_end <= true;
        wait;
    end process;

    process(s_clk)
        variable v_cnt        : natural := 0;
        variable v_hs_low     : natural := 0;
        variable v_line_len   : natural := 0;
        variable v_act_pix    : natural := 0;
        variable v_lines      : natural := 0;
        variable v_vs_low_l   : natural := 0;
        variable v_act_lines  : natural := 0;
        variable v_prev_hs    : std_logic := '1';
        variable v_prev_vs    : std_logic := '1';
        variable v_line_act   : boolean := false;
        variable v_checked_h  : boolean := false;
        variable v_checked_v  : natural := 0;
        variable v_max_addr   : natural := 0;
        variable v_frame_cyc  : natural := 0;
        variable v_max_x, v_max_y : natural := 0;
        variable v_min_addr   : natural := natural'high;
    begin
        if rising_edge(s_clk) and s_rst_n = '1' then
            v_line_len := v_line_len + 1;
            v_frame_cyc := v_frame_cyc + 1;

            if s_hs = '0' then
                v_hs_low := v_hs_low + 1;
            end if;

            if s_pixel_en = '1' then
                v_act_pix := v_act_pix + 1;
                v_line_act := true;

                if s_addr > v_max_addr then
                    v_max_addr := s_addr;
                end if;

                if s_addr < v_min_addr then
                    v_min_addr := s_addr;
                end if;

                if s_x > v_max_x then
                    v_max_x := s_x;
                end if;

                if s_y > v_max_y then
                    v_max_y := s_y;
                end if;
            end if;

            if v_prev_hs = '1' and s_hs = '0' then
                if v_lines > 2 and not v_checked_h then
                    report "Ligne : " & integer'image(v_line_len) & " cycles, hs bas : "
                         & integer'image(v_hs_low) & ", pixels actifs : " & integer'image(v_act_pix);

                    assert v_line_len = 858
                        report "Erreur : ligne /= 858 cycles"
                        severity error;

                    assert v_hs_low = 62
                        report "Erreur : hs bas /= 62 cycles"
                        severity error;

                    v_checked_h := true;
                end if;

                if s_vs = '0' then
                    v_vs_low_l := v_vs_low_l + 1;
                end if;

                if v_line_act then
                    v_act_lines := v_act_lines + 1;
                end if;

                v_lines := v_lines + 1;
                v_line_len := 0;
                v_hs_low := 0;
                v_act_pix := 0;
                v_line_act := false;
            end if;

            if v_prev_vs = '1' and s_vs = '0' then
                if v_checked_v = 2 then
                    report "Image : " & integer'image(v_frame_cyc) & " cycles "
                         & integer'image(v_frame_cyc/858) & " lignes), lignes actives : "
                         & integer'image(v_act_lines) & ", x max : " & integer'image(v_max_x)
                         & ", y max : " & integer'image(v_max_y) & ", adresse min/max : "
                         & integer'image(v_min_addr) & " / " & integer'image(v_max_addr);

                    assert v_frame_cyc = 858*525
                        report "Erreur : image /= 525 lignes"
                        severity error;

                    assert v_act_lines = 480
                        report "Erreur : lignes actives /= 480"
                        severity error;

                    assert v_max_x = 719 and v_max_y = 479
                        report "Erreur : x/y max"
                        severity error;

                    assert v_min_addr = 0 and v_max_addr = 720*480-1
                        report "Erreur : adresse"
                        severity error;

                    report "Fin de simulation : timings conformes"
                        severity note;
                end if;

                v_checked_v := v_checked_v + 1;
                v_lines := 0;
                v_vs_low_l := 0;
                v_act_lines := 0;
                v_max_addr := 0;
                v_frame_cyc := 0;
                v_max_x := 0;
                v_max_y := 0;
                v_min_addr := natural'high;
            end if;

            v_prev_hs := s_hs;
            v_prev_vs := s_vs;
        end if;
    end process;

end architecture sim;
```

# Synchronisation horizontale
<img width="1917" height="1115" alt="synchroHorizontale" src="https://github.com/user-attachments/assets/73250f1d-51df-41a7-8d1d-1b165bafcbf9" />

# Synchronisation verticale
<img width="1917" height="1111" alt="visualisationVerticale" src="https://github.com/user-attachments/assets/dda84ffd-f227-447f-b884-98c65ac501bf" />

### Étape 3 : Instanciation du contrôleur HDMI

Le composant hdmi_controler est instancié avec l'horloge 27 MHz et le reset produits par la PLL.
Les sorties o_pixel_en et o_pixel_address ne sont pas utilisées.

```vhdl
hdmi_controler0 : entity work.hdmi_controler
    port map (
        i_clk           => s_clk_27,
        i_rst_n         => s_rst_n,
        o_hdmi_hs       => o_hdmi_tx_hs,
        o_hdmi_vs       => o_hdmi_tx_vs,
        o_hdmi_de       => o_hdmi_tx_de,
        o_pixel_en      => open,
        o_pixel_address => open,
        o_x_counter     => s_x_counter,
        o_y_counter     => s_y_counter
    );

o_hdmi_tx_clk <= s_clk_27;
```

# RTLViewer
<img width="1917" height="1103" alt="rtlViewer (2)" src="https://github.com/user-attachments/assets/65fac419-e959-468c-9114-04603e82c5e7" />

# Question 4
### Affichage d'un dégradé et test sur la carte

Les sorties pixel o_hdmi_tx_d ont été reliées aux compteurs x et y du contrôleur :

```vhdl
o_hdmi_tx_d(23 downto 16) <= std_logic_vector(to_unsigned(s_x_counter, 8));
o_hdmi_tx_d(15 downto 8)  <= std_logic_vector(to_unsigned(s_y_counter, 8));
o_hdmi_tx_d(7 downto 0)   <= (others => '0');
```

Après compilation et programmation de la carte, l'image a été observée dans VLC grâce à l'adaptateur HDMI-USB.

<img width="667" height="622" alt="alwen" src="https://github.com/user-attachments/assets/1e35b585-9920-4886-8da3-4a40b0e50f60" />


L'image est stable et découpée en 3 × 2 carreaux. to_unsigned(…, 8) ne garde que les 8 bits de poids faible : la couleur repart de 0 tous les 256 pixels, alors que x va jusqu'à 719 et y jusqu'à 479.

*À quels bits correspond chaque composante couleur ?*

Le rouge augmente de gauche à droite (comme x), le vert de haut en bas (comme y), et le bleu n'apparaît jamais. En bas à droite de chaque carreau, rouge et vert sont au maximum, ce qui donne du jaune.

| Bits de o_hdmi_tx_d | Composante |
|---|---|
| 23 – 16 | Rouge |
| 15 – 8  | Vert  |
| 7 – 0   | Bleu  |

Le pixel est donc codé en RGB sur 24 bits, avec 8 bits par couleur.

## Déplacement d'un pixel

### Principe

Le contrôleur HDMI balaie l'écran pixel par pixel. À chaque instant, il fournit la position du pixel en cours d'affichage (s_x_counter, s_y_counter). Les deux encodeurs donnent la position du point à afficher :
- l'encodeur *gauche* pour x (déplacement horizontal) ;
- l'encodeur *droit* pour y (déplacement vertical).



Quand le pixel balayé correspond à la position des encodeurs, on envoie du *blanc* (x"FFFFFF"). Sinon, on envoie du *noir* (x"000000").



### 1. Modification de telecran.vhd

*Deux encodeurs.* Le composant encodeur de la première partie est instancié deux fois, avec l'horloge et le reset de la PLL. La sortie de l'encodeur gauche est aussi envoyée sur les LED, pour visualiser la position x en binaire.

```vhdl
encoder_left : entity work.encoder
    generic map (G_WIDTH => 10, G_CLK_HZ => 27_000_000,
                 G_SAMPLE_US => 1_000, G_MAX => h_res - C_SIZE)
    port map (i_clk => s_clk_27, i_rst_n => s_rst_n,
              i_a => i_left_ch_a, i_b => i_left_ch_b, o_value => s_enc_x);
o_leds <= s_enc_x;

encoder_right : entity work.encoder
    generic map (G_WIDTH => 10, G_CLK_HZ => 27_000_000,
                 G_SAMPLE_US => 1_000, G_MAX => v_res - C_SIZE)
    port map (i_clk => s_clk_27, i_rst_n => s_rst_n,
              i_a => i_right_ch_a, i_b => i_right_ch_b, o_value => s_enc_y);
```

*Affichage du point.*

```vhdl
constant C_SIZE : natural := 4;  
...
o_hdmi_tx_d <= x"FFFFFF" when (s_x_counter >= to_integer(unsigned(s_enc_x)))
                          and (s_x_counter <  to_integer(unsigned(s_enc_x)) + C_SIZE)
                          and (s_y_counter >= to_integer(unsigned(s_enc_y)))
                          and (s_y_counter <  to_integer(unsigned(s_enc_y)) + C_SIZE)
               else x"000000";
```

# RTLViewer 
<p align="center">
<img width="1912" height="1087" alt="RTLDeplacement" src="https://github.com/user-attachments/assets/9243129d-2255-461e-be0f-38b7453a0564" />
  <br><em>Figure : RTL Viewer — deux encodeurs, le contrôleur HDMI et la logique de comparaison vers <code>o_hdmi_tx_d</code></em>
</p>

# Problème rencontré : le point qui saute

Avec l'encodeur de la première partie, un seul cran de la molette faisait parfois avancer le point de plusieurs pixels.
Cela vient des *rebonds mécaniques* de l'encodeur : à chaque changement d'état, le contact oscille brièvement
et produit plusieurs fronts au lieu d'un seul. Lus à 27 MHz, ces rebonds étaient comptés comme plusieurs crans.

*Solution :* nous avons ajouté un diviseur dans encoder.vhd. Il génère une impulsion s_en toutes les *1 ms*,
et les signaux A et B ne sont lus qu'à ces instants. Les rebonds, beaucoup plus courts, sont ainsi ignorés.

```vhdl
if (s_en = '1') then
    r_a1 <= i_a;  r_a2 <= r_a1;
    r_b1 <= i_b;  r_b2 <= r_b1;
end if;
```


### 2. Test sur la carte


| Action | Résultat observé |
|---|---|
| Appui sur KEY0 | Le point revient dans le coin haut gauche (peu visible au bord de l'image), les LED s'éteignent |
| Rotation de l'encodeur gauche | Le point se déplace horizontalement, les LED affichent x |
| Rotation de l'encodeur droit | Le point se déplace verticalement |
| Rotation lente, cran par cran | Déplacement régulier, sans saut |



Vidéo de démonstration :


https://github.com/user-attachments/assets/8f1063c5-5501-4a82-b22c-aca2886b4cf1



## Mémorisation

Pour garder le dessin à l'écran, comme sur un vrai écran magique, il faut mémoriser tous les pixels déjà parcourus. Pour cela, on utilise un framebuffer : une mémoire qui contient une case par pixel de l'écran, réalisée avec la RAM dual-port fournie (dpram.vhd).

### 1. Qu'est-ce qu'une mémoire dual-port ?

Une mémoire RAM dual-port est *une seule mémoire* qui possède *deux ports d'accès indépendants*, A et B. Chaque port a sa propre horloge, sa propre adresse, ses propres données et son propre signal d'écriture. On peut donc, *au même moment*, écrire une case par un port et lire une autre case par l'autre port.

Ports de dpram.vhd (identiques pour A et B) :

| Signal | Rôle |
|---|---|
| i_clk_a / i_clk_b | Horloge du port |
| i_addr_a / i_addr_b | Adresse de la case (0 à mem_size − 1) |
| i_data_a / i_data_b | Donnée à écrire |
| i_we_a / i_we_b | Autorisation d'écriture (1 = écriture) |
| o_q_a / o_q_b | Donnée lue, disponible au coup d'horloge suivant |

La mémoire est *synchrone* : tout se passe sur le front montant de l'horloge. Les deux ports accèdent à la même variable ram, donc une donnée écrite par le port A peut être lue par le port B. Seul cas à éviter : les deux ports qui écrivent en même temps à la même adresse.



