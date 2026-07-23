  package com.sunucu.auth;

import org.bukkit.ChatColor;
import org.bukkit.command.Command;
import org.bukkit.command.CommandExecutor;
import org.bukkit.command.CommandSender;
import org.bukkit.entity.Player;

import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

public class AuthSystem implements CommandExecutor {

    // Oyuncu verilerini tutan geçici veri yapıları (Gerçek projede Veritabanı/YAML kullanılır)
    private final Map<UUID, String> registeredPlayers = new HashMap<>();
    private final Map<UUID, Boolean> loggedInPlayers = new HashMap<>();
    private final Map<UUID, Integer> loginAttempts = new HashMap<>();
    private final Map<UUID, Long> joinTime = new HashMap<>();

    // Ayarlar
    private final int MIN_PASSWORD_LENGTH = 6;
    private final int MAX_PASSWORD_LENGTH = 16;
    private final int LOGIN_TIMEOUT_SECONDS = 30; // 30 saniye içinde giriş yapmazsa uyarır/atabilir
    private final int MAX_ATTEMPTS = 3; // Maksimum hatalı şifre hakkı

    @Override
    public boolean onCommand(CommandSender sender, Command command, String label, String[] args) {
        if (!(sender instanceof Player)) {
            sender.sendMessage("❌ Bu komutu sadece oyuncular kullanabilir!");
            return true;
        }

        Player player = (Player) sender;
        UUID uuid = player.getUniqueId();
        String cmdName = command.getName().toLowerCase();

        // -------------------------------------------------------------
        // 1. /kayit <şifre> <şifreTekrar> KOMUTU
        // -------------------------------------------------------------
        if (cmdName.equals("kayit")) {
            if (registeredPlayers.containsKey(uuid)) {
                player.sendMessage(ChatColor.RED + "❌ Zaten bir hesabınız var! Lütfen " + ChatColor.YELLOW + "/gir <şifre>" + ChatColor.RED + " komutunu kullanın.");
                return true;
            }

            if (args.length < 2) {
                player.sendMessage(ChatColor.YELLOW + "⚠️ Kullanım: " + ChatColor.WHITE + "/kayit <şifre> <şifreTekrar>");
                return true;
            }

            String password = args[0];
            String passwordConfirm = args[1];

            // Şifre Uzunluk Kontrolü
            if (password.length() < MIN_PASSWORD_LENGTH || password.length() > MAX_PASSWORD_LENGTH) {
                player.sendMessage(ChatColor.RED + "❌ Şifreniz en az " + MIN_PASSWORD_LENGTH + " ve en fazla " + MAX_PASSWORD_LENGTH + " karakter olmalıdır!");
                return true;
            }

            // Şifre Eşleşme Kontrolü
            if (!password.equals(passwordConfirm)) {
                player.sendMessage(ChatColor.RED + "❌ Girdiğiniz şifreler birbirleriyle eşleşmiyor!");
                return true;
            }

            // Kayıt Başarılı
            registeredPlayers.put(uuid, password); // Not: Gerçek projede şifreler hash'lenerek saklanmalıdır (örn: BCrypt)
            loggedInPlayers.put(uuid, true);
            player.sendMessage(ChatColor.GREEN + "✅ Başarıyla kayıt oldunuz ve giriş yapıldı! 🔑");
            player.sendMessage(ChatColor.AQUA + "🛡️ Sunucumuza hoş geldiniz, keyifli oyunlar!");
            return true;
        }

        // -------------------------------------------------------------
        // 2. /gir <şifre> KOMUTU
        // -------------------------------------------------------------
        if (cmdName.equals("gir")) {
            if (!registeredPlayers.containsKey(uuid)) {
                player.sendMessage(ChatColor.RED + "❌ Henüz kayıtlı değilsiniz! Lütfen " + ChatColor.YELLOW + "/kayit <şifre> <şifreTekrar>" + ChatColor.RED + " kullanın.");
                return true;
            }

            if (loggedInPlayers.getOrDefault(uuid, false)) {
                player.sendMessage(ChatColor.GREEN + "✅ Zaten giriş yapmış durumdasınız!");
                return true;
            }

            // Süre (Timeout) Kontrolü
            long elapsedSeconds = (System.currentTimeMillis() - joinTime.getOrDefault(uuid, System.currentTimeMillis())) / 1000;
            if (elapsedSeconds > LOGIN_TIMEOUT_SECONDS) {
                player.sendMessage(ChatColor.RED + "⏱️ Giriş yapmak için ayrılan " + LOGIN_TIMEOUT_SECONDS + " saniyelik süreniz doldu!");
                player.kickPlayer(ChatColor.RED + "⏱️ Giriş yapma süreniz dolduğu için sunucudan atıldınız.");
                return true;
            }

            if (args.length < 1) {
                player.sendMessage(ChatColor.YELLOW + "⚠️ Kullanım: " + ChatColor.WHITE + "/gir <şifre>");
                return true;
            }

            String inputPassword = args[0];
            String actualPassword = registeredPlayers.get(uuid);

            // Şifre Doğru
            if (inputPassword.equals(actualPassword)) {
                loggedInPlayers.put(uuid, true);
                loginAttempts.remove(uuid); // Başarılı girişte deneme sayısını sıfırla
                player.sendMessage(ChatColor.GREEN + "✅ Başarıyla giriş yaptınız! Hoş geldiniz 🔓");
            } else {
                // Şifre Yanlış - Hatalı Deneme Takibi
                int attempts = loginAttempts.getOrDefault(uuid, 0) + 1;
                loginAttempts.put(uuid, attempts);

                if (attempts >= MAX_ATTEMPTS) {
                    player.kickPlayer(ChatColor.RED + "❌ " + MAX_ATTEMPTS + " kez üst üste yanlış şifre girdiğiniz için sunucudan atıldınız! 🛡️");
                } else {
                    int remaining = MAX_ATTEMPTS - attempts;
                    player.sendMessage(ChatColor.RED + "❌ Yanlış şifre! Kalan deneme hakkınız: " + ChatColor.GOLD + remaining + " ⚠️");
                }
            }
            return true;
        }

        return false;
    }

    // Oyuncu sunucuya katıldığında çağrılacak metot (Event Listener içine eklenebilir)
    public void onPlayerJoin(Player player) {
        UUID uuid = player.getUniqueId();
        loggedInPlayers.put(uuid, false);
        joinTime.put(uuid, System.currentTimeMillis());

        player.sendMessage("--------------------------------------------------");
        if (registeredPlayers.containsKey(uuid)) {
            player.sendMessage(ChatColor.GOLD + "🔑 Lütfen hesabınıza giriş yapın: " + ChatColor.YELLOW + "/gir <şifre>");
            player.sendMessage(ChatColor.GRAY + "⏱️ Giriş yapmak için " + LOGIN_TIMEOUT_SECONDS + " saniyeniz var.");
        } else {
            player.sendMessage(ChatColor.LIGHT_PURPLE + "📝 Hesabınızı oluşturun: " + ChatColor.YELLOW + "/kayit <şifre> <şifreTekrar>");
        }
        player.sendMessage("--------------------------------------------------");
    }
}
